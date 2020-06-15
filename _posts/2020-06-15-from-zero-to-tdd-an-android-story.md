---
title: "From Zero to TDD: An Android-story"
layout: post
---

# From Zero to TDD: An Android-story

{{ page.date | date_to_string }}

I've had a special relationship to automated testing during my career as a developer. At times I've been
quite fond of it, and occasionally have practiced test-driven development. Recently, as I've been focusing
my career on Android development, I have not been doing any significant amount of testing, as I've
struggled a lot with getting good value out of UI testing, and the remaining parts of our application have
been fairly simple.

However, more recently, I've been inspired by my colleagues to try my hand at actually writing tests again,
and perhaps even adopt a TDD-workflow. And, I'm happy to say, I have succeeded.

## The tech-stack

- JUnit

On the JVM, this is likely what you are going to be using when it comes to writing tests.

- Espresso

UI-tests on Android are written using Espresso, which plays a similar role to Selenium in web development.
Espresso allows for UI actions to be automated, and UI assertions to be performed.

- Kakao

Given that we are using Kotlin, we can utilize some nice tooling to get a DSL for Espresso actions/assertions.
Writing them for hand is not the most pleasant experience imaginable.

- Apollo Idling Resource

The application I'm working on uses the excellent GraphQL client `apollo-android` almost exclusively for
performing API interactions. Since it has native support for idle callbacks, which then allows us to add an
Espresso Idling Resource only in test code, we'll be using it to make our tests know to wait until network
calls are done before performing any interactions/assertions.

- MockWebServer

I'm generally not a fan of tests making real API calls. On the application I'm working on, this would also be
essentially impossible due to third-party integrations utilized. Instead, we use MockWebServer to provide the
possible responses our APIs would hand out to us.

- Awaitility

Sometimes, an assertion cannot be made instantly. Maybe there's an animation running, that you were unable to
turn off due to the animation disabling in Espresso-tests not being universal. Perhaps you do not want to
introduce Idling Resource-semaphores in your production code, as that would introduce test code in your
production code. In these cases, you can instead use Awaitility as a DSL to describe await conditions.

## First test, no network calls

First off, we need some scaffolding:

```kotlin
@RunWith(AndroidJUnit4::class)
class ExampleActivityTest {
    @get:Rule
    val activityRule = ActivityTestRule(ExampleActivity::class.java)
}
```

The `@RunWith`-annotation makes it so that we run this as an instrumented test on an Android Device/Emulator.

The `activityRule` specifies which Activity we intend to test.

Since we're using Kakao, we're also going to define a `Screen`:

```kotlin
class ExampleScreen: Screen<ExampleScreen>() {
    val loadingSpinner = KView { withId(R.id.loadingSpinner) }
}
```

I'm not entirely sure why I need to pass the Screen-class itself as a type parameter, but whatever.

Finally, we want an actual test inside our `ExampleActivityTest`-class:

```kotlin
// ...
    @Test
    fun shouldShowLoadingSpinnerInitially() {
        onScreen<ExampleScreen> {
            loadingSpinner { isVisible() }
        }
    }
// ...
```

## Introducing some network into the mix

Most applications need to do some form of API interations to actually be useful. Architecturally, our app has three primary layers:

- View (Activities, Fragments and Custom Views)
- ViewModel (provides the View-layer with data that can be observed)
- Data-layer (provides the ViewModel with data, mostly from network but potentially from other sources, such as local cache)

We stitch this together using Dependency Injection, for which we use Koin.

In our data-layer, we use an `ApolloClient` to make API calls to our GraphQL server.

To actually make this work, we unfortunately need to make some minor changes to our production code. Bear with me.

This was our application class:

```kotlin
class App: Application() {
    // ...
}
```

What we needed to do was this:

```kotlin
open class App: Application() {
    open val graphqlUrl = "https://graphql.example.com/graphql"
}
```

This enables us to add the following scaffolding:

```kotlin
class TestApp: App() {
    override val graphqlUrl = "http://localhost:8080"
}

class TestRunner: AndroidJUnitRunner() {
    override fun newApplication(cl: ClassLoader?, className: String?, context: Context?): Application
        = super.newApplication(cl, TestApp::class.java.name, context)
}
```

And don't forget to add this newly minted `TestRunner` to your `build.gradle`-file, or it won't work:

```groovy
android {
    defaultConfig {
        testInstrumentationRunner "dev.oscarnylander.tddsample.TestRunner"
    }
}
```

As we also want our Espresso actions/assertions to wait while we make network calls, we also want to take this opportunity
to set up our Apollo Idling Resource:

```kotlin
// ...

    private val apolloClient: ApolloClient by inject()

    override fun onCreate() {
        super.onCreate()

        val idlingResource = ApolloIdlingResource.create("ApolloIdlingResource", apolloClient)
        IdlingRegistry
            .getInstance()
            .register(idlingResource)
    }
// ...
```

Finally, we can now write a test with mocked network calls:

```kotlin
// ...
    @get:Rule
    val activityRule = ActivityTestRule(
        ExampleActivity::class.java,
        false,
        false // Do not launch the activity right away, so that we have time to mock
    )

    @Test
    fun shouldShowDataOnScreen() {
        MockWebServer().use { webServer ->
            webServer.start(8080)
            webServer.enqueue(MockResponse().setBody(DATA.toJson()))

            activityRule.launchActivity(null)

            onScreen<ExampleScreen> {
                film { hasText("The Empire Strikes Back") }
            }
        }
    }

    companion object {
        // Data class generated by `apollo-android`, based on your `.graphql`-files
        private val DATA = FilmQuery.Data(
            film = FilmQuery.Film(
                title = "The Empire Strikes Back"
            )
        )
    }
// ...
```

## Oh no! A scary animation

Animations are great. They make your app look good, but also provide an enhanced user experience if you
use the animations to visually explain to your user what is going on in your app. I personally think
that for a good user experience, good animations are mandatory. However, animations are tricky when
it comes to UI testing, as they introduce another dimension to your assertions: time. When animating,
the final outcome is generally not available instantly, but instead after a delay while the animation
is running.

For example, this test:

```kotlin
// ...
    @Test
    fun shouldDoSomethingWhenClicked() {
        // ...
        onScreen<ExampleScreen> {
            button { click() }
            viewShownAfterAnimation {
                isVisible()
            }
        }
        // ...
    }
// ...
```

will fail in a scenario where there is an animation before `viewShownAfterAnimation` actually becomes visible.

There are two recommended approachs to solve this issue when making Espresso tests:

1. Disable animations while running Espresso tests.
This is not necessarily a bad idea - animations take time, which makes your test execution take more time.
However, this only works for some types of animations. Two notable exceptions are `ViewPropertyAnimator`-based
animations, and `SpringAnimation`s. This is a problem for us, as we use these two types of animations quite
frequently throughout our app, ruling out this option.

2. Use `IdlingResource`s in production code.
I'm not fond of this option at all. Test code should really not be present in your production code,
this is a recipe for bad outcomes.

I found a third approach, which I prefer: Use `awaitility`!

`awaitility` is a DSL that helps you express asynchronous behaviour when testing. With it, we can fix the
previously mentioned test, as such:

```kotlin
// ...
    @Test
    fun shouldDoSomethingWhenClicked() {
        // ...
        onScreen<ExampleScreen> {
            button { click() }
            await atMost ONE_SECOND untilAsserted {
                viewShownAfterAnimation {
                    isVisible()
                }
            }
        }
        // ...
    }
// ...
```

Now we allow the animation one second to complete before considering this test to fail. Nice!

## The devil is in the (shared) state

Time passes, and you've written a bunch of tests. You decide to add them to CI, so that you can ensure that
the behaviours you want the application to have stays the same over time. But, for some unknown reason,
the tests fail when you run all of them at the same time. In isolation, they all run fine. What could be wrong?

The answer in my case was the fact that the application does not restart between test executions, and hence,
there may be some shared state between test executions. More specifically, it was the fact that
the Apollo Client caches the HTTP responses per query, leading the the wrong data being loaded when multiple
tests mocked responses for the same query.

The solution was fairly straight-forward: Reset the state between test executions. You may solve this using
Koins testing facilities, and the JUnit `@Before`-annotation:

```kotlin
class ExampleActivityTest: KoinTest {
    private val apolloClient: ApolloClient by inject()

    @Before
    fun setup() {
        apolloClient.clearNormalizedCache()
    }
}
```

After adding this method, the tests now run fine even when all tests are ran one after each other.

## Fin

This was everything I needed to go from Zero to TDD on Android. I'm quite pleased with the results, so far.
It's going to be interesting to see how this holds up over time.

You can find a sample project here: <https://github.com/hedvigoscar/android-apollo-tdd-sample>
