+++
title = "Introduction to Model View Presenter on Android"
draft = false
date = "2015-03-23"
tags = ["Android"]
+++

This article is a step-by-step introduction to MVP on Android, from a simplest possible
example to best practices. The article also introduces a new library
that makes MVP on Android extremely simple.

<!--more-->

# Is it simple? How can I benefit of using it?

## What is MVP

* **View** is a layer that displays data and reacts to user actions.
On Android, this could be an Activity, a Fragment, an android.view.View or a Dialog.
* **Model** is a data access layer such as database API or remote server API.
* **Presenter** is a layer that provides View with data from Model.
Presenter also handles background tasks.

On Android MVP is a way to separate background tasks from activities/views/fragments
to make them independent of most lifecycle-related events. This way an
application becomes simpler, overall application reliability increases up to 10 times,
application code becomes shorter, code maintainability becomes better
and developer's life becomes happier.

## Why MVP on Android

### Reason 1: Keep It Stupid Simple

If you haven't read this article yet, do it: [The Kiss Principle](
http://web.archive.org/web/20160206225831/https://people.apache.org/~fhanik/kiss.html)

* Most of the modern Android applications just use View-Model architecture.
* Programmers are involved into fight with View complexities instead of solving business tasks.

Using only Model-View in your application you usually end up with "everything is connected with everything".

![](/images/mvp_everything_is_connected_with_everything.png)

If this diagram does not look complex, then think about each View
can disappear and appear at random time. Do not forget about saving/restoring of Views.
Attach a couple of background tasks to that temporary Views, and the cake is ready!

An alternative to the "everything is connected with everything" is a god object.

![](/images/mvp_a_god_object.png)

A god object is overcomplicated; its parts cannot be reused, tested or easily debugged and refactored.

With MVP

![](/images/mvp_mvp.png)

* Complex tasks are split into simpler tasks and are easier to solve.
* Smaller objects, less bugs, easier to debug.
* Testable.

View layer with MVP becomes so simple, so it does not even need to have callbacks
when requesting for data. View logic becomes very linear.

### Reason 2: Background tasks

Whenever you write an Activity, a Fragment or a custom View, you can put all
methods that are connected with
background tasks to a different external or static class. This way your background tasks will not be
connected with an Activity, will not leak memory and will not depend on Activity's
recreation. We call such object "Presenter".

There are few different approaches to handle background tasks,
a properly implemented MVP library is one of the most reliable.

#### **Why this works**

Here is a little diagram that shows what happens with different application parts
during a configuration change or during an out-of-memory event. Every Android developer should know
this data, however this data is surprisingly hard to find.

```
                                          |    Case 1     |   Case 2     |    Case 3
                                          |A configuration| An activity  |  A process
                                          |   change      |   restart    |   restart
 ---------------------------------------- | ------------- | ------------ | ------------
 Dialog                                   |     reset     |    reset     |    reset
 Activity, View, Fragment                 | save/restore  | save/restore | save/restore
 Fragment with setRetainInstance(true)    |   no change   | save/restore | save/restore
 Static variables and threads             |   no change   |   no change  |    reset
```

**Case 1**: A configuration change normally happens when a user flips the screen,
changes language settings, attaches an external monitor, etc. More on this event you can read
here: [configChanges](http://developer.android.com/reference/android/R.attr.html#configChanges).

**Case 2**: An Activity restart happens when a user has set "Don't keep activities" checkbox in Developer's settings
and another activity becomes topmost.

**Case 3**: A process restart happens if there is not enough memory and the application is in the background.

**Conclusion**

Now you can see, a Fragment with setRetainInstance(true) does not help here - we need
to save/restore such fragment's state anyway. So we can simply throw away retained fragments
to limit the number of problems. 

```
                                          |A configuration|
                                          |   change,     |
                                          | An activity   |  A process
                                          |   restart     |   restart
 ---------------------------------------- | ------------- | -------------
 Activity, View, Fragment, DialogFragment | save/restore  | save/restore
 Static variables and threads             |   no change   |    reset
```

Now it looks much better. We only need to write two pieces of code to *completely* restore an application
in any possible case:

* save/restore for Activity, View, Fragment, DialogFragment;

* restart background requests in case of a process restart.

The first part can done by usual means of Android API. The second part is a job for Presenter.
Presenter just remembers which requests it should execute, and if a process restarts during execution,
Presenter will execute them again.

# A simple example

This example will load and display some items from remote server. If an error occurs
a little toast will be shown.

I recommend [RxJava](https://github.com/ReactiveX/RxJava) usage to build presenters because this library
allows to control data flows easily.

I would like to thank the guy who created a simple api that I use for my examples: [The Internet Chuck Norris Database](http://www.icndb.com/)

## Without MVP [example 00](https://github.com/konmik/MVPExamples/tree/master/example00):

``` java
public class MainActivity extends Activity {
    public static final String DEFAULT_NAME = "Chuck Norris";

    private ArrayAdapter<ServerAPI.Item> adapter;
    private Subscription subscription;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ListView listView = (ListView)findViewById(R.id.listView);
        listView.setAdapter(adapter = new ArrayAdapter<>(this, R.layout.item));
        requestItems(DEFAULT_NAME);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unsubscribe();
    }

    public void requestItems(String name) {
        unsubscribe();
        subscription = App.getServerAPI()
            .getItems(name.split("\\s+")[0], name.split("\\s+")[1])
            .delay(1, TimeUnit.SECONDS)
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new Action1<ServerAPI.Response>() {
                @Override
                public void call(ServerAPI.Response response) {
                    onItemsNext(response.items);
                }
            }, new Action1<Throwable>() {
                @Override
                public void call(Throwable error) {
                    onItemsError(error);
                }
            });
    }

    public void onItemsNext(ServerAPI.Item[] items) {
        adapter.clear();
        adapter.addAll(items);
    }

    public void onItemsError(Throwable throwable) {
        Toast.makeText(this, throwable.getMessage(), Toast.LENGTH_LONG).show();
    }

    private void unsubscribe() {
        if (subscription != null) {
            subscription.unsubscribe();
            subscription = null;
        }
    }
}
```

An experienced developer will notice that this simple example has some
critical defects in it:

* A request starts every time a user flips the screen - an app makes more requests
than needed and the user observes an empty screen for a moment after each screen flip.

* If a user flips the screen often this will cause memory leaks - every callback
keeps a reference to MainActivity and will keep it in memory while a request is running.
It is absolutely possible to get an application crash because of out-of-memory error or
a significant application slowdown.

## With MVP [example 01](https://github.com/konmik/MVPExamples/tree/master/example01):

Please, do not try this at home! :) This example is for demonstration purposes only.
In the real life you're not going to use a static variable to hold your presenter.

``` java
public class MainPresenter {

    public static final String DEFAULT_NAME = "Chuck Norris";

    private ServerAPI.Item[] items;
    private Throwable error;

    private MainActivity view;

    public MainPresenter() {
        App.getServerAPI()
            .getItems(DEFAULT_NAME.split("\\s+")[0], DEFAULT_NAME.split("\\s+")[1])
            .delay(1, TimeUnit.SECONDS)
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new Action1<ServerAPI.Response>() {
                @Override
                public void call(ServerAPI.Response response) {
                    items = response.items;
                    publish();
                }
            }, new Action1<Throwable>() {
                @Override
                public void call(Throwable throwable) {
                    error = throwable;
                    publish();
                }
            });
    }

    public void onTakeView(MainActivity view) {
        this.view = view;
        publish();
    }

    private void publish() {
        if (view != null) {
            if (items != null)
                view.onItemsNext(items);
            else if (error != null)
                view.onItemsError(error);
        }
    }
}
```

Technically speaking, MainPresenter has three "streams" of events: *onNext*, *onError*, *onTakeView*.
They join in `publish()` method and *onNext* or *onError* values become published to a MainActivity instance
that has been supplied with *onTakeView*.

``` java
public class MainActivity extends Activity {

    private ArrayAdapter<ServerAPI.Item> adapter;

    private static MainPresenter presenter;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ListView listView = (ListView)findViewById(R.id.listView);
        listView.setAdapter(adapter = new ArrayAdapter<>(this, R.layout.item));

        if (presenter == null)
            presenter = new MainPresenter();
        presenter.onTakeView(this);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        presenter.onTakeView(null);
        if (!isChangingConfigurations())
            presenter = null;
    }

    public void onItemsNext(ServerAPI.Item[] items) {
        adapter.clear();
        adapter.addAll(items);
    }

    public void onItemsError(Throwable throwable) {
        Toast.makeText(this, throwable.getMessage(), Toast.LENGTH_LONG).show();
    }
}
```

MainActivity creates MainPresenter and keeps it outside of reach of onCreate/onDestroy cycle. MainActivity uses
a static variable to reference MainPresenter, so every time a process restarts due to out-of-memory event,
MainActivity should check if the presenter is still here and create it if needed.

Yes, this looks a little bit bloated with checks and uses a static variable, but later I will show
how to make this look much better. :)

The main idea is:

* The example application does not start a request every time a user flips the screen.

* If a process has been restarted the example loads data again.

* MainPresenter does not keep a reference to a MainActivity instance while MainActivity is destroyed,
so there is no memory leak on a screen flip, and there is no need to unsubscribe the request.

# [Nucleus](https://github.com/konmik/nucleus)

Nucleus is a library I created while was inspired by [Mortar](https://github.com/square/mortar) library
and [Keep It Stupid Simple](https://people.apache.org/~fhanik/kiss.html) article.

Here is a list of features:

* It supports save/restore Presenter's state in a View/Fragment/Activity's state Bundle. A Presenter can save request arguments
  into that bundles to restart them later.

* It provides a facility to direct request results and errors right into a view
  with just one line of code, so you don't have to write all that `!= null` checks.

* It allows you to have more than one instance of a view that requires a presenter.
  You can't do this if you're instantiating a presenter with [Dagger](http://square.github.io/dagger/) (a traditional way).

* It provides a shortcut for binding a presenter to a view with just one line.

* It provides base View classes: `NucleusView`, `NucleusFragment`, `NucleusSupportFragment`, `NucleusActivity`. You can also copy/paste code from one
  of them to make any class you use to utilize presenters of Nucleus.

* It can automatically restart requests after a process restart and automatically unsubscribe RxJava subscriptions during `onDestroy`.

* And finally, it is plain simple, so any developer can understand it. (I recommend to spend some time diving into RxJava though.)
  There are just about 180 lines of code to drive Presenter and 230 lines of code for RxJava support.

## Example with [Nucleus](https://github.com/konmik/nucleus) [example 02](https://github.com/konmik/MVPExamples/tree/master/example02)

Note: since the moment of writing this article, a new version of Nucleus was released. You can find updated examples in the [Nucleus project repository](https://github.com/konmik/nucleus).

``` java
public class MainPresenter extends RxPresenter<MainActivity> {

    public static final String DEFAULT_NAME = "Chuck Norris";

    @Override
    protected void onCreate(Bundle savedState) {
        super.onCreate(savedState);

        App.getServerAPI()
            .getItems(DEFAULT_NAME.split("\\s+")[0], DEFAULT_NAME.split("\\s+")[1])
            .delay(1, TimeUnit.SECONDS)
            .observeOn(AndroidSchedulers.mainThread())
            .compose(this.<ServerAPI.Response>deliverLatestCache())
            .subscribe(new Action1<ServerAPI.Response>() {
                @Override
                public void call(ServerAPI.Response response) {
                    getView().onItemsNext(response.items);
                }
            }, new Action1<Throwable>() {
                @Override
                public void call(Throwable throwable) {
                    getView().onItemsError(throwable);
                }
            });
    }
}

@RequiresPresenter(MainPresenter.class)
public class MainActivity extends NucleusActivity<MainPresenter> {

    private ArrayAdapter<ServerAPI.Item> adapter;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ListView listView = (ListView)findViewById(R.id.listView);
        listView.setAdapter(adapter = new ArrayAdapter<>(this, R.layout.item));
    }

    public void onItemsNext(ServerAPI.Item[] items) {
        adapter.clear();
        adapter.addAll(items);
    }

    public void onItemsError(Throwable throwable) {
        Toast.makeText(this, throwable.getMessage(), Toast.LENGTH_LONG).show();
    }
}
```

As you can see, this example is significantly shorter and cleaner than the previous one. Nucleus creates/destroys/saves presenter,
attaches/detaches a View to it and sends request results right into an attached view automatically.

`MainPresenter`'s code is shorter because
it uses `deliverLatestCache()` operation that delays all data and errors that has been emitted by a data source
until a view becomes available. It also caches data in memory so it can be reused on configuration change.

`MainActivity`'s code is shorter because presenter's creation is managed by `NucleusActivity`. All you need to
bind a presenter is to write `@RequiresPresenter(MainPresenter.class)` annotation.

Warning! An annotation! In Android world, if you use annotations, it is a good practice to check that this
will not degrade performance. The benchmark I've made on *Galaxy S* (year 2010 device) says that processing this annotation
takes less that 0.3 ms. This happens only during instantiation of a view, so the annotation is considered to be free.

## More examples

An extended example with request arguments persistence is here: [Nucleus Example](https://github.com/konmik/nucleus/tree/master/nucleus-example).

An example with unit tests: [Nucleus Example With Tests](https://github.com/konmik/nucleus/tree/master/nucleus-example-with-tests)

## `deliverLatestCache()` method

This RxPresenter helping method has three variants:

* `deliver()` will just delay all onNext, onError and onComplete emissions until a View becomes available.
 Use it for cases when you're doing a one-time request, like logging in to a web service.

* `deliverLatest()` will drop the older onNext value if a new onNext value is available. If you have an updatable source
 of data this will allow you to not accumulate data that is not necessary.

* `deliverLatestCache()` is the same as `deliverLatest()` but in addition it will keep the latest result in memory
 and will re-deliver it when another instance of a view becomes available (i.e. on configuration change). If you don't want to
 organize save/restore of a request result in your view (in case if a result is big or it can not be easily saved into Bundle) this
 method will allow you to make user experience better.

## Presenter's lifecycle

Presenter's lifecycle is significantly shorter that a lifecycle of other Android components.

* `void onCreate(Bundle savedState)` - is called on every presenter's creation.

* `void onDestroy()` - is called when user a View becomes destroyed not because of configuration change.

* `void onSave(Bundle state)` - is called during View's `onSaveInstanceState` to persist Presenter's state as well.

* `void onTakeView(ViewType view)` - is called during Activity's or Fragment's `onResume()`, or during `android.view.View#onAttachedToWindow()`.

* `void onDropView()` - is called during Activity's `onDestroy()` or Fragment's `onDestroyView()`, or during `android.view.View#onDetachedFromWindow()`.

## View recycling and view stack

Normally your views (i.e. fragments and custom views) are attached and detached randomly during user interactions.
This could be a good idea to not destroy a presenter every time a view is detached. You can detach and attach
views any time and presenters will outlive all this actions, continuing background work.

# Best practices

## Save your request arguments inside Presenter

The rule is simple: the presenter's main purpose is to manage requests. So View should not handle or restart requests itself.
From a View's perspective, background tasks are something that never disappear and will always return a result or an error *without
any callbacks*.

``` java
public class MainPresenter extends RxPresenter<MainActivity> {

    private String name = DEFAULT_NAME;

    @Override
    protected void onCreate(Bundle savedState) {
        super.onCreate(savedState);
        if (savedState != null)
            name = savedState.getString(NAME_KEY);
        ...

    @Override
    protected void onSave(@NonNull Bundle state) {
        super.onSave(state);
        state.putString(NAME_KEY, name);
    }
```

I recommend using awesome [Icepick](https://github.com/frankiesardo/icepick) library. It reduces code size and simplifies
application logic without using runtime annotations - everything happens during compilation. This library is
a good partner for [ButterKnife](http://jakewharton.github.io/butterknife).

``` java
public class MainPresenter extends RxPresenter<MainActivity> {

    @State String name = DEFAULT_NAME;

    @Override
    protected void onCreate(Bundle savedState) {
        super.onCreate(savedState);
        Icepick.restoreInstanceState(this, savedState);
        ...

    @Override
    protected void onSave(@NonNull Bundle state) {
        super.onSave(state);
        Icepick.saveInstanceState(this, state);
    }
```

If you have more than a couple of request arguments this library naturally saves life. You can create `BasePresenter` and put *Icepick*
into that class once and all subclasses will automatically get the ability to save their fields that are annotated
with `@State` and you will never need to implement `onSave` again. This also works for saving Activity's, Fragment's or View's state.

## Execute instant queries on the main thread in `onTakeView`

Sometimes you have a short data query, such as reading a small amount of data from a database. While you can easily
create a restartable request with Nucleus, you don't have to use this powerful tool everywhere. If you're
initiating a background request during a fragment's creation, a user will see a blank screen for a moment, even if the query will take just
a couple of milliseconds. So, to make code shorter and users happier, use the main thread.

## Do not try to make your Presenter control your View

This does not work well - the application logic becomes too complex because it goes in an unnatural way.

The natural way is to make a flow of control to go from a user, through view, presenter and model to data. In the end a user
will use the application and the user is a source of control for the application. So control should go from a user rather than from
a some internal application structure.

When control goes from View to Presenter and then from Presenter to Model it is just a direct flow, it is easy to write code
like this. You get an easy **user -> view -> presenter -> model -> data** sequence.
But when control goes like this: **user -> view -> presenter -> view -> presenter -> model -> data**, it is
just violates KISS principle. Don't play ping-pong between your view and presenter.

# Conclusion

Give MVP a try, tell a friend. :)
