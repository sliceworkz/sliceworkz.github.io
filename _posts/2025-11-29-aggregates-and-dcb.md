---
layout: post
toc: true
title: Aggregates and DCB
description: Aggregates and Dynamic Consistency Boundary
date: 2025-11-29 07:00:00
categories: [Eventstore Documentation,Eventstore Application Scenarios]
tags: [aggregate,dcb,dynamic consistency boundary]
---
# Aggregates and Dynamic Consistency Boundaries

This guide explores the aggregate pattern from DDD tactical design and how Dynamic Consistency Boundaries (DCB) offer a flexible alternative for managing consistency in event-sourced systems.
While there are many ways to implement Aggregates, we focus on event sourced applications implemented on the Sliceworkz Eventstore library, so all what follows is written from that perspective.

## Aggregate Concept

An **aggregate** is a tactical design pattern from Domain-Driven Design (DDD) that structures applications around consistency boundaries. The **aggregate root** is the entry point for all operations on the aggregate.

Key characteristics:
- **Consistency boundary**: All invariants within the aggregate are checked atomically
- **Transactional boundary**: Changes to the aggregate produce events in a single transaction
- **Identity**: Each aggregate instance has a unique identifier (the aggregate root ID)
- **Event production**: Commands check invariants and produce events to append to the event stream

In the EventStore library, you can aggregates can have their own EventStream (e.g., `EventStreamId.forContext("students").withPurpose("123")`

But another option is to put all events in one stream, use the `Tag` concept to link them to business concepts, and build the Aggregates on those:
- **Tags**: To mark events as belonging to a specific aggregate instance (e.g., `Tags.of("student", "123")`)
- **EventStreamId**: To separate different aggregate types (e.g., `EventStreamId.forContext("students")`)

```java
// Query all events for aggregate instance "123"
EventQuery query = EventQuery.forEvents(
    EventTypesFilter.any(),
    Tags.of("student", "123")
);
```

## Code Example: Student

A Student aggregate that manages registration, name changes, and unsubscription:

```java
sealed interface LearningDomainEvent {}

sealed interface StudentDomainEvent extends LearningDomainEvent {
    record StudentRegistered(String name) implements StudentDomainEvent {}
    record StudentNameChanged(String name) implements StudentDomainEvent {}
    record StudentUnsubscribed() implements StudentDomainEvent {}
    // Inherits StudentSubscribedToCourse from RegistrationEvents
}

class Student implements EventWithMetaDataHandler<StudentDomainEvent> {
    String studentId;
    String name;
    boolean active;
    EventReference lastEventReference;

    public Student(String studentId) {
        this.studentId = studentId;
    }

    // Command handlers - check invariants and produce events
    public List<StudentDomainEvent> register(String name) {
        if (active) {
            throw new IllegalStateException("Student already registered");
        }
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name cannot be empty");
        }
        return List.of(new StudentDomainEvent.StudentRegistered(name));
    }

    public List<StudentDomainEvent> changeName(String newName) {
        if (!active) {
            throw new IllegalStateException("Student not active");
        }
        if (name.equals(newName)) {
            return List.of(); // No change
        }
        return List.of(new StudentDomainEvent.StudentNameChanged(newName));
    }

    public List<StudentDomainEvent> unsubscribe() {
        if (!active) {
            throw new IllegalStateException("Student already unsubscribed");
        }
        return List.of(new StudentDomainEvent.StudentUnsubscribed());
    }

    // Event handlers - apply state changes
    private void when (StudentDomainEvent event) {
        switch(event) {
            case StudentDomainEvent.StudentRegistered r -> {
                this.name = r.name();
                this.active = true;
            }
            case StudentDomainEvent.StudentNameChanged n -> {
                this.name = n.name();
            }
            case StudentDomainEvent.StudentUnsubscribed u -> {
                this.active = false;
            }
        }
    }

    @Override
    public void when(Event<StudentDomainEvent> event) {
        when(event.data());
        this.lastEventReference = event.reference();
    }

    public EventReference lastEventReference() {
        return lastEventReference;
    }
}
```

**Loading and saving**:

```java
// Load aggregate from events
Student loadStudent(String studentId) {
    Student student = new Student(studentId);
    EventQuery query = EventQuery.forEvents(
        EventTypesFilter.any(),
        Tags.of("student", studentId)
    );
    stream.query(query)
        .forEach(event -> student.when(event.cast()));
    return student;
}

// Save events with optimistic locking
void saveStudent(Student student, List<StudentDomainEvent> events) {
    stream.append(
        AppendCriteria.of(
            EventQuery.forEvents(
                EventTypesFilter.any(),
                Tags.of("student", student.studentId)
            ),
            student.lastEventReference()
        ),
        events.stream()
            .<EphemeralEvent<? extends LearningDomainEvent>>map(e -> Event.of(e, Tags.of("student", student.studentId)))
            .toList()
    );
}
```

## Code Example: Course

A Course aggregate managing course definition, capacity, and cancellation can be created in the exact same way:

```java
sealed interface CourseDomainEvent extends LearningDomainEvent {
    record CourseDefined(String name, int capacity) implements CourseDomainEvent {}
    record CourseCapacityUpdated(int newCapacity) implements CourseDomainEvent {}
    record CourseCancelled() implements CourseDomainEvent {}
    // Inherits StudentSubscribedToCourse from RegistrationEvents
}

class Course implements EventWithMetaDataHandler<CourseDomainEvent> {
    String courseId;
    String name;
    int capacity;
    boolean active;
    EventReference lastEventReference;

    public Course(String courseId) {
        this.courseId = courseId;
    }

    public List<CourseDomainEvent> define(String name, int capacity) {
        if (active) {
            throw new IllegalStateException("Course already defined");
        }
        if (capacity <= 0) {
            throw new IllegalArgumentException("Capacity must be positive");
        }
        return List.of(new CourseDomainEvent.CourseDefined(name, capacity));
    }

    public List<CourseDomainEvent> updateCapacity(int newCapacity) {
        if (!active) {
            throw new IllegalStateException("Course not active");
        }
        if (newCapacity <= 0) {
            throw new IllegalArgumentException("Capacity must be positive");
        }
        if (this.capacity == newCapacity) {
            return List.of();
        }
        return List.of(new CourseDomainEvent.CourseCapacityUpdated(newCapacity));
    }

    public List<CourseDomainEvent> cancel() {
        if (!active) {
            throw new IllegalStateException("Course already cancelled");
        }
        return List.of(new CourseDomainEvent.CourseCancelled());
    }

    private void when(CourseDomainEvent event) {
        switch(event) {
            case CourseDomainEvent.CourseDefined d -> {
                this.name = d.name();
                this.capacity = d.capacity();
                this.active = true;
            }
            case CourseDomainEvent.CourseCapacityUpdated u -> {
                this.capacity = u.newCapacity();
            }
            case CourseDomainEvent.CourseCancelled c -> {
                this.active = false;
            }
        }
    }

    @Override
    public void when(Event<CourseDomainEvent> event) {
        when(event.data());
        this.lastEventReference = event.reference();
    }

    public EventReference lastEventReference() {
        return lastEventReference;
    }
}
```

**Loading and saving**:

```java
// Load aggregate from events
Course loadCourse(String courseId) {
    Course course = new Course(courseId);
    EventQuery query = EventQuery.forEvents(
        EventTypesFilter.any(),
        Tags.of("course", courseId)
    );
    stream.query(query)
        .forEach(event -> course.when(event.cast()));
    return course;
}

// Save events with optimistic locking
void saveCourse(Course course, List<CourseDomainEvent> events) {
    stream.append(
        AppendCriteria.of(
            EventQuery.forEvents(
                EventTypesFilter.any(),
                Tags.of("course", course.courseId)
            ),
            course.lastEventReference()
        ),
        events.stream()
            .<EphemeralEvent<? extends LearningDomainEvent>>map(e -> Event.of(e, Tags.of("course", course.courseId)))
            .toList()
    );
}
```

## Limitations of Aggregates

Consider extending these examples with a subscription concept where:
- A student can subscribe to **maximum 5 courses**
- A course cannot be subscribed to if it has reached its **maximum capacity**

This creates a problem: **the consistency boundary crosses aggregate boundaries**.

**Attempt 1 - Check in Student aggregate**:
```java
// Student needs to know about subscriptions
public List<StudentEvents> subscribe(String courseId) {
    if (subscriptionCount >= 5) {
        throw new IllegalStateException("Maximum 5 courses");
    }
    // But we can't check if the course has capacity!
}
```

**Attempt 2 - Check in Course aggregate**:
```java
// Course needs to know about subscriptions
public List<CourseEvents> acceptSubscription(String studentId) {
    if (currentSubscriptions >= capacity) {
        throw new IllegalStateException("Course full");
    }
    // But we can't check if student already has 5 courses!
}
```

Both invariants require information from **both** aggregates, yet aggregates are designed to be independent consistency boundaries. You're stuck.

## Aggregates or Dynamic Consistency Boundaries?

Sara Pellegrini's articles series starting from ["I am here to kill the aggregate"](https://sara.event-thinking.io/2023/04/kill-aggregate-chapter-1-I-am-here-to-kill-the-aggregate.html) challenge the fixed consistency boundary of aggregates, and offer an elegant solution called Dynamic Consistency Boundaries (DCB).
If you don't yet fully understand or appreciate the brilliant combination of simplicity and power found in what she proposes, do yourself a favor and go read her articles and watch her talks.

We'll limit ourselves to a brief practical application in an example here:

The **aggregate boundary is fixed** by design—which works well in many situations but becomes problematic when:
- Business rules span multiple aggregates
- Consistency requirements are dynamic based on business context
- You need flexible consistency boundaries

**Three Options for Cross-Aggregate Invariants**:

1. **Merge aggregates**: Combine Student and Course into a single "LearningManagement" aggregate
   - Feels wrong—it's purely a technical solution
   - Loses domain clarity
   - Creates a "God aggregate"

2. **Use a Saga pattern**: Coordinate between aggregates via process managers
   - Complicates the solution
   - No grounding in business domain
   - Eventual consistency where immediate consistency is needed
   - Complex compensation logic

3. **Use DCB**: Check invariants in a joint consistency boundary
   - Query relevant facts from both concepts
   - Check all invariants together
   - Append events atomically with optimistic locking
   - Natural business operation

**Combining Aggregates and DCB**:

Once you're into DCB, chances are you will kill your aggregates (all of them).  Let's for now assume in this example we'll only use it in a complementary way when we hit the limitations of our Aggregates.

## Code Example: Combining Aggregates & DCB

Define a new Domain Event to register course subscriptions:

```java
sealed interface RegistrationDomainEvent extends LearningDomainEvent {
    record StudentSubscribedToCourse(String studentId, String courseId)
        implements RegistrationDomainEvent {}
}
```

**RegistrationDecisionModel**:

Implement a specific readmodel that will act as a usecase-specific decision model.
It will consume exactly the events that are needed to come up with the right business decision:

```java
class RegistrationDecisionModel implements EventHandler<LearningDomainEvent> {

	private String studentId;
	private String courseId;

	private boolean studentAlreadySubscribed;
	private int studentSubscriptions;
	private int courseSubsciptions;
	private int courseCapacity;

	public RegistrationDecisionModel ( String studentId, String courseId ) {
		this.studentId = studentId;
		this.courseId = courseId;
	}

	public EventQuery getEventQuery ( ) {
		// this query will deliver all events linked to the student at hand, including subscriptions to other courses
		EventQuery studentQuery = EventQuery.forEvents(EventTypesFilter.of(StudentSubscribedToCourse.class), Tags.of("student", studentId));

		// this query will deliver all events linked to the course at hand, including subscriptions from other students
		EventQuery courseQuery = EventQuery.forEvents(EventTypesFilter.of(CourseDefined.class, CourseCapacityUpdated.class, StudentSubscribedToCourse.class), Tags.of("course", courseId));

		// ask for all matching events (union query)
		return studentQuery.combineWith(courseQuery);
	}


	@Override
	public void when(LearningDomainEvent event) {
		switch ( event ) {
			case RegistrationDomainEvent.StudentSubscribedToCourse s -> {
				if ( s.studentId().equals(studentId) ) {
					studentSubscriptions++;
				}
				if ( s.courseId().equals(courseId) ) {
					courseSubsciptions++;
				}
				if ( s.studentId().equals(studentId) && s.courseId().equals(courseId) ) {
					studentAlreadySubscribed = true;
				}
			}
			case CourseDomainEvent.CourseDefined d -> {
				courseCapacity = d.capacity();
			}
			case CourseDomainEvent.CourseCapacityUpdated u -> {
				courseCapacity = u.newCapacity();
			}
			default -> {
				// not of interest to our decision
			}
		}
	}

	public boolean canSubscribe ( ) {
		boolean result = true;
		if ( studentAlreadySubscribed ) {
			System.out.println("already subscribed to this course");
			result = false; // student is already subscribed to this course
		}
		if ( studentSubscriptions >= 5 ) {
			System.out.println("student already subscribed to 5 courses");
			result = false; // student is already subscribed to 5 (other) courses
		}
		if ( courseSubsciptions >= courseCapacity ) {
			System.out.println("course already at capacity");
			result = false; // course has already reached its maximum capacity
		}
		return result;
	}

}
```

**Service using DCB**:

```java
public boolean subscribeStudentToCourse(String studentId, String courseId) {

    RegistrationDecisionModel dm = new RegistrationDecisionModel(studentId, courseId);
    // remark: in practice, we would use additional decision models eg to determine if the studentId and the courseId exist at all.

    List<Event<LearningDomainEvent>> relevantEvents = stream.query(dm.getEventQuery()).toList();
    EventReference lastRef = relevantEvents.getLast().reference();
    relevantEvents.forEach(dm::when);

    if ( dm.canSubscribe() ) {
        stream.append(
                AppendCriteria.of(dm.getEventQuery(), lastRef),
                Event.of(
                    new StudentSubscribedToCourse(studentId, courseId),
                    Tags.of(Tag.of("student", studentId), Tag.of("course", courseId))
                )
            );
        return true;
    } else {
    	// can't subscribe; get the details, throw an error, ...
    	return false;
    }

}
```

**Key Benefits**:

- **Natural business operation**: Subscription is modeled as a first-class concept
- **Aggregates remain cohesive**: Student and Course don't need to know about each other
- **Cross-aggregate consistency**: Both invariants checked atomically via DCB
- **Flexible boundaries**: The consistency boundary is dynamic based on the operation

The combination of aggregates for internal consistency and DCB for cross-aggregate consistency provides the best of both worlds.

Of course, one could also consume the Subscription events in either aggregate as well, in order to support other aggregate-local decisions, eg:
- avoid that the capacity of a Course is reduced below the existing subscription count
- avoid that a Student unsubscribes from our application with pending subscriptions for upcoming courses
- etc...