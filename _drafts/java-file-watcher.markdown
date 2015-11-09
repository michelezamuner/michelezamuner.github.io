---
layout: post
title:  "Java File Watcher"
categories: jekyll update
---
While working on a little personal project called Groph, which basically consists in a small Web application that manages documents' references through tags and categories, I stumbled upon some interesting issues regarding the synchronization between a Java application using the `WatchService` framework (Java 7 and above), and the filesystem.

I'm currently working on a small component capable of registering files to watch, and then notifying the external world about the occurred events. My idea is to use it to build a configuration component that automatically updates itself every time configuration files change, much like what the `Preferences` framework does.

Obviously, I could actually have used the `Preferences` framework in the first place. However I didn't like all [the work](http://www.davidc.net/programming/java/java-preferences-using-file-backing-store) involved with setting a custom configuration file, plus the fact that apparently [this is not how preferences are supposed to be used](http://stackoverflow.com/questions/7947672/java-preferences-api-using-a-custom-file). Furthermore, most importantly, I wanted to delve deeper into some lower level details about the Java ecosystem, and the point of pretty much all this project is to practice Java development...

Let's start with a pretty basic example of usage of the `WatchService` framework:

{% highlight java %}
// This contains the WatchService framework,
// amongst other things...
import java.nio.file.*;
// Statically import constants representing all the
// possible kinds of events.
import static java.nio.file.StandardWatchEventKinds.*;

// [...]

// This is the main service that retrieves filesystem events
/** @throws IOException **/
WatchService service = FileSystems.getDefault().newWatchService();

// As an example, we are watching the current directory.
Path directory = Paths.get(".");

// Path is a Watchable, so we can register a WatchService to it,
// providing the kinds of events we want to watch.
/** @throws IOException **/
directory.register(service, ENTRY_CREATE, ENTRY_MODIFY);

// We need to continuously watch over filesystem events
while (true)
{
    // This put the current thread in a wait-like state,
    // until the filesystem notifies that a new bunch of
    // events occurred. The returned key contains references
    // to the current events, and other related information.
    /** @throws InterruptedException **/
    WatchKey key = service.take();

    // Cycle over the events carried by the current key.
    for (WatchEvent<?> event : key.pollEvents())
    {
        // This is an exceptional situation that happens
        // when too many events are generated, and too few
        // are consumed, and in other special cases.
        if (event.kind() == OVERFLOW) continue;

        // Downcast the event object from the generic type
        // to the specific Path type. See below to known
        // why this is always safe.
        @SuppressWarnings("unchecked")
        WatchEvent<Path> ev = (WatchEvent<Path>)event;

        // The context() method return a Path representing
        // the actual file the event is related to.
        System.out.println(ev.context());
    }

    // Reset the key, so it can be used again to refer to
    // the next bunch of events. If this fails, it means
    // that for some reason the key is not available any more.
    if (!key.reset())
        throw new IOException("Invalid key");
}
{% endhighlight %}
