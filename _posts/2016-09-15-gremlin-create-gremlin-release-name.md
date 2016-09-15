---
title: Using Gremlin to Create Gremlin Release Name
layout: post
---

Marko said:
{% highlight text %}
The 3.x line has the following naming conventions: 
1. Has to use the word “Gremlin" in the release name. 
2. Has to have a number in the release name. 
3. Has to be music related.  
 
Thus,  
3.0: A Gremlin Raga in 7/16 Time 
3.1: A 187 on the Undercover Gremlinz 
3.2: Nine Inch Gremlins 3.3: ???
{% endhighlight %}

So let's use Gremlin to help create its next release name.

First, we need [songs with numbers in the title](http://www.songfacts.com/category-songs_with_numbers_in_the_title.php)  to satisfy criteria 3.  I took that web page and extracted the song title, artist into a csv file.

Now we can start gremlin console.

We'll need a good CSV parser to parse this file because it has quoted fields thus a simple split(',') won't do.

In gemlin console:
{% highlight text %}
:install  com.xlson.groovycsv groovycsv 1.1
import com.xlson.groovycsv.CsvParser
import java.util.stream.Stream;
{% endhighlight %}

Now we'll create the graph, even though this isn't graphy,  and load the data

{% highlight groovy %}
graph = TinkerGraph.open()
g = graph.traversal()

csv = new File("songs.csv").text

data = new CsvParser().parse(csv, readFirstLine:true, columnNames:['title', 'artist'])
for(line in data) {
   g.addV(label,"song", "title", line.title.toString(), "artist", line.artist).iterate()
}
{% endhighlight %}

Now we need to create a list of 'numbers', both digits and spelled-out, for criteria 2 and a list of nouns that can be replaced with 'Gremlin' for criteria 1.

{% highlight groovy %}
 words = g.V().values('title').map({s -> s.get().split(' ').toList()}).unfold().toList()
 new File('numbers.txt').withWriter{ out -> words.each { out.println it } }
 new File('words.txt').withWriter{ out -> words.each { out.println it } }
 {% endhighlight %}
 
 Then painstakingly go through each list and remove what doesn't belong.  Once done, you can load the data.
 
 {% highlight groovy %}
 new File("words.txt").each({ line ->
  g.addV(label, "noun", "word", line).iterate()
})

new File("numbers.txt").each({ line ->
  g.addV(label, "numbers", "value", line).iterate()
})
{% endhighlight %}

Now that all the data is loaded, we can use gremlin to create the next great release name!

Let's also include a non-deterministic random generator.

{% highlight groovy %}
import java.security.SecureRandom

textContainsAny = { List z -> new P({x, y -> 
  boolean hasIt = false
  y.each { v -> 
    if (x.contains(v))
      hasIt=true;
    }
  return hasIt; }, z)
}

replaceNounWithGremlin = { title, nouns -> 
  results = []
  nouns.each { n ->
    if (title.contains(n)) {
      results << title.replace(n, "Gremlin")
    }
  }
  return results;
}

numbers = g.V().hasLabel("numbers").values('value').toList()

words = g.V().hasLabel("noun").values('word').toList()

gremlins = g.V().hasLabel("song").
             has('title', textContainsAny(numbers)).
             values('title').map({it -> replaceNounWithGremlin(it.get(), words)}).
             unfold().dedup().toList()
													 
Collections.shuffle(gremlins, new SecureRandom())

gremlins.first()

{% endhighlight %}

I have included the data files and a ready-to-go groovy script:

* [songs.csv](/songs.csv)
* [numbers.txt](/numbers.txt)
* [words.txt](/words.txt)
* [release-name-generator.groovy](/release-name-generator.groovy)


