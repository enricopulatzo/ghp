Mocking Inaccessible Fields in Java
===================================

Here's the scenario:
- You've got a component that handles exceptions for your module.
- You're using [SLF4J](http://slf4j.org/) and [JUnit 4](http://junit.org/junit4/).
- You're tasked with ensuring that certain fields are in message logged when an exception is thrown.
- Your organization insists on a high degree of mutation coverage in order to keep your tests honest.
- For the sake of example, let's pretend you don't have any ability to refactor the production code.
- Write a test that verifies that the desired fields are logged.

```
public class SomeComponent {
  private static final Logger LOGGER = LoggerFactory.getLogger(SomeComponent.class);
  public void doThingWhichThrowsAnException() {
    try {
      client.attemptServiceCall();
    } catch (SomeBusinessException ex) {
      LOGGER.warn("I cannot complete this operation", ex);
    }
  }
}
```

Looking at the above code, your mutation coverage tool will likely remove the `LOGGER.warn()` call leaving you with one more mutant to kill. There's really only one solution at this time: you've got to muck with the `LOGGER` field.

Since the field in question is declared `private static final` one may use either reflection or a bytecode library to manipulate the field at runtime.

Let's use reflection to manipulate the field, and we'll replace the value of `LOGGER` with a mock object created by [Mockito](http://site.mockito.org/).

```
@Test
public void ensureLoggerCallIsMade() throws Exception {
  Field field = SomeComponent.class.getDeclaredField("LOGGER");
  field.setAccessible(true);
  Field modifiersField = Field.class.getDeclaredField("modifiers");
  modifiersField.setAccessible(true);
  modifiersField.setInt(field, field.getModifiers() & ~Modifier.FINAL);
  Logger mockLogger = mock(Logger.class);
  field.set(null, mockLogger);
  
  clientShouldThrowExceptionOnAttempt(); // this method is left as an exercise to the reader
  
  fixture.doThingWhichThrowsAnException();
  
  verify(mockLogger).warn(eq("I cannot complete this operation"), isA(SomeBusinessException.class)); 
}
```

Now this code sort of works, but has one glaring problem: we don't clean up the logger field, leaving other test code quite potentially broken. This is bad news. Not only that, but most of our test is messing around with the logger field, and not really setting up our test. Finally, if one has to do a similar refletion-based mock field swap in any other code, some redundancies are going to occur.

Fortunately, JUnit (starting with 4.7 for some reason) introduced the concept of `TestRule`s. I like to think of `TestRule` as an encapsulation of a `@Before` and `@After` method call.

Let's rework our refection usage into a `TestRule` class so we can swap out any `private static final` field in any code we come across (unless there's a Security Manager in place, then you'll need some other blog posts).

```
public class MockitoPrivateStaticFieldRule<TYPE> implements TestRule {
	private static final String MODIFIERS = "modifiers";
	private final Class<?> classToModify;
	private final String fieldName;
	private Object originalItem;
	TYPE replacementItem;
	public TYPE getMockObject() {
		return replacementItem;
	}

	public MockitoPrivateStaticFieldRule(final Class<?> classToModify, final String fieldName, final Class<TYPE> mockType) {
		this.classToModify = classToModify;
		this.fieldName = fieldName;
		replacementItem = Mockito.mock(mockType);
	}
  
  @Override
	public Statement apply(final Statement base, final Description description) {
		return new Statement() {
			@Override
			public void evaluate() throws Throwable {
        Mockito.reset(replacementItem);
				captureField();
        try {
  				base.evaluate();
        } finally {
  				restoreField();
        }
			}
		};
	}

	private Object updateField(final Class<?> classToModify, final String fieldName, final Integer modifierMask, final Object newValue) throws Exception {
		final Field field = classToModify.getDeclaredField(fieldName);
		field.setAccessible(true);
		Field modifiersField = Field.class.getDeclaredField(MODIFIERS);
		modifiersField.setAccessible(true);
		modifiersField.setInt(field, field.getModifiers() & modifierMask);
		modifiersField.setAccessible(false);
    final Object result = field.get(null);
		field.set(null, newValue);
    field.setAccessible(false);
		return result;
	}

	private void captureField() throws Exception {
		originalItem = updateField(classToModify, fieldName, ~Modifier.FINAL, replacementItem);
	}

	private void restoreField() throws Exception {
		updateField(classToModify, fieldName, Modifier.FINAL, originalItem);
	}
}
```

The constructor of our class takes three arguments: a class to modify, the field name to be modified, and the type that field happens to be. The constructor also creates our mock object using Mockito. A getter exists to access the mock object in a `@Test`.

The value of the `TestRule` is that JUnit runs it for us, transparent to our code. The `captureField()` method makes our field accessible for just long enough to be replaced with a mock object. We stash the original value for later. The `TestRule` then allows our `@Test` to run, and finally we clean up the field by restoring the original object.

Our test class looks like this:
```
@Rule
public MockitoPrivateStaticFieldRule<Logger> loggerRule = new MockitoPrivateStaticFieldRule<>(SomeComponent.class, "LOGGER", Logger.class);

@Test
public void ensureLoggerCallIsMade() throws Exception {
  Logger mockLogger = loggerRule.getMockObject();
  
  clientShouldThrowExceptionOnAttempt(); // this method is left as an exercise to the reader
  
  fixture.doThingWhichThrowsAnException();
  
  verify(mockLogger).warn(eq("I cannot complete this operation"), isA(SomeBusinessException.class)); 
}
```

Now our test class is much cleaner and when we want to replicate this method in another class, we simply instantiate another MockitoPrivateStaticFieldRule. One thing I really like about this technique is it allows you to modify any class, not just the class under test. One can also employ multiple MockitoPrivateStaticFieldRule instances to swap out as many fields are necessary.
