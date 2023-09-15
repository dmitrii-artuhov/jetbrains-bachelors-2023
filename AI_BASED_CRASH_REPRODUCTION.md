# AI-based Crash Reproduction

## Task 1:
> Why are you interested in this project and how do you see its implementation?

### Motivation: 

I am enthusiastic about participating in this research work for several compelling reasons that align with my background and interests in Java and Kotlin programming. 

I have a genuine passion for software development, and I understand the significance of robust and reliable software systems. My knowledge in Java and my ongoing learning journey in Kotlin have provided me with a solid foundation in two related to the research work programming languages. This knowledge base equips me with the skills necessary to deliver new code, understand already written codebases, and contribute effectively to the research.

The integration of AI techniques, particularly Large Language Models, into software development tasks has piqued my interest. The potential of AI in generating tests and reproducing complex crashes represents a cutting-edge approach that holds great promise. I am eager to dive into this field and contribute to its advancement.

More about my prior experience and my projects you can see in my resume [here](https://drive.google.com/file/d/1xAeLxKeC-u-VbqWDPDTSS_Y4QYsHiI0N/view?usp=sharing).

### Implementation thoughts:

How I see the implementation of this project.

1. Application of LLMs for crash reproduction:
    
    a. It should work independently:
    
    - We could use the prompt generating approach that was introduced in the [Exploring LLM-based General Bug Reproduction](https://arxiv.org/pdf/2209.11515.pdf) paper, specifically using markdown with some sections (stack trace of the crash, crash description if exists). For the source code, the number of tokens might be overwhelming and most of the code will be unrelated to crash reproduction. To handle that we might only send source code for those classes that are reachable from the method calls in the stack trace. Handling special method calls (e.g. native calls for read/write operations) might require further investigation. We want LLM to generate multiple answers with some differences, for that we have to tune some model parameters.
    - In the paper you provided (as I understood) no source code was fed to the LLM thats why algorithm for dependencies resolution was provided. Even though we feed the source code, problems with dependency resolution might still occur. We could enhance the *Algorithm 1* provided in the paper in order to handle that.
    - If no valid candidate tests for crash reproduction were found then we could take the best candidates (the heuristics for picking "the best" candidates are described in *Algorithm 2* of the paper) and generate new prompt for the LLM feeding those tests and the reasons why they are not working (e.g. stack trace is different, no crash appeared, etc.) and ask the LLM to regenerate tests.

    b. Works as a helper for other crash reproduction techniques:

    - We might ask LLM to generate the tests with prompt that I described in bullet-point (a) and then feed those tests to the evolutionary-based crash reproduction algorithm. Another use case might be to use LLM on the mutation stage of evolutionary-based algorithm, because LLM might generate some mutations that might not be achievable with using it.

2. Assess the availability of crash reproduction in real-world cases:
    - For that we can aggregate the existing stack traces reported for Intellij IDEA, generate prompts for them and run the newly created LLM model on each prompt in both variants: as a standalone crash reproduction tool and as a helper for the existing algorithms. 



## Task 2:

> Reproduce the crashes.

### Crash #1

```java
java.lang.ArrayIndexOutOfBoundsException: 4
    at org.apache.commons.lang3.time.FastDateParser.toArray(FastDateParser.java:413)
    at org.apache.commons.lang3.time.FastDateParser.getDisplayNames(FastDateParser.java:381)
    at org.apache.commons.lang3.time.FastDateParser$TextStrategy.addRegex(FastDateParser.java:664)
    at org.apache.commons.lang3.time.FastDateParser.init(FastDateParser.java:138)
    at org.apache.commons.lang3.time.FastDateParser.<init>(FastDateParser.java:108)
    at org.apache.commons.lang3.time.FastDateFormat.<init>(FastDateFormat.java:370)
    at org.apache.commons.lang3.time.FastDateFormat$1.createInstance(FastDateFormat.java:91)
    at org.apache.commons.lang3.time.FastDateFormat$1.createInstance(FastDateFormat.java:88)
    at org.apache.commons.lang3.time.FormatCache.getInstance(FormatCache.java:82)
    at org.apache.commons.lang3.time.FastDateFormat.getInstance(FastDateFormat.java:165)
```

#### Solution:

My JUnit-4.10 test that reproduces the same exception and the same stack trace:

```java
@Test(expected = ArrayIndexOutOfBoundsException.class)
public void testApacheCommonsLang() {
    String pattern = "Ga"; // use Era designator (eg. AD, BC). Make sure that there are designators after the 'G', (otherwise, the same exception will be thrown with the same reason, but from the different line)
    TimeZone timeZone = TimeZone.getDefault();
    Locale locale = FastDateParser.JAPANESE_IMPERIAL; // use this locale to utilize Japanese imperial calendar
    FastDateFormat.getInstance(pattern, timeZone, locale);
}
```

If I print the thrown exception:

```java
java.lang.ArrayIndexOutOfBoundsException: 5
	at org.apache.commons.lang3.time.FastDateParser.toArray(FastDateParser.java:413)
	at org.apache.commons.lang3.time.FastDateParser.getDisplayNames(FastDateParser.java:381)
	at org.apache.commons.lang3.time.FastDateParser$TextStrategy.addRegex(FastDateParser.java:664)
	at org.apache.commons.lang3.time.FastDateParser.init(FastDateParser.java:138)
	at org.apache.commons.lang3.time.FastDateParser.<init>(FastDateParser.java:108)
	at org.apache.commons.lang3.time.FastDateFormat.<init>(FastDateFormat.java:370)
	at org.apache.commons.lang3.time.FastDateFormat$1.createInstance(FastDateFormat.java:91)
	at org.apache.commons.lang3.time.FastDateFormat$1.createInstance(FastDateFormat.java:88)
	at org.apache.commons.lang3.time.FormatCache.getInstance(FormatCache.java:82)
	at org.apache.commons.lang3.time.FastDateFormat.getInstance(FastDateFormat.java:165)
```

There is a difference in the exception message probably because of the platform and setup differences. I used (in both crash reproductions): 
- Windows 10
- Amazon corretto 1.8 SDK
- Java 8


#### Explanation:

1. First of all I was able to reproduce this bug the way I specified only with the Japanese Imperial locale (which is `new Locale("ja", "JP", "JP") => ja_JP_JP_#u-ca-japanese`)
2. The exception is thrown from `FastDateParser(String pattern, TimeZone timeZone, Locale locale)`, the error occurs when the short regex is being built for the specified pattern in `init` method:
    ```java
    ...
    currentFormatField = patternMatcher.group();
    Strategy currentStrategy = getStrategy(currentFormatField);
    for(;;) {
        patternMatcher.region(patternMatcher.end(), patternMatcher.regionEnd());
        if(!patternMatcher.lookingAt()) {
            nextStrategy = null;
            break;
        }
        String nextFormatField= patternMatcher.group();
        nextStrategy = getStrategy(nextFormatField);
        if(currentStrategy.addRegex(this, regex)) {
            // short regex is accumulated in `regex` variable every time `currentStrategy.addRegex(...)` method is called
            collector.add(currentStrategy);
        }
        currentFormatField = nextFormatField;
        currentStrategy = nextStrategy;
    }
    if(currentStrategy.addRegex(this, regex)) { // here as well
        collector.add(currentStrategy);
    }
    ...
    ```
    According to the stack trace, exception is thrown from `FastDateParser$TextStrategy.addRegex(...)` method. Because of this we are interested only in those strategy classes that are the instance of `TextStrategy` class, which are: `ERA_STRATEGY`, `DAY_OF_WEEK_STRATEGY`, `AM_PM_STRATEGY`, and `TEXT_MONTH_STRATEGY`.

    Looking at method `FastDateParser::getStrategy(String formatField)` we see that patterns that might produce such exception must contain: *'G', 'E', 'a', 'M'* (label meanings are explained [here](https://docs.oracle.com/javase/8/docs/api/java/text/SimpleDateFormat.html)). Testing for all of them showed that the pattern that causes the exception includes `G` (which states for era label).

3.  Method `TextStrategy::addRegex(...)` calls `FastDateParser::getDisplayNames(int)` in its body with `Calendar.ERA` parameter. Then `Calendar::getDisplayNames(...)` is called, which returns the map of available era labels for the specified locale and their indexes. In my setup it returned: `{H=4, M=1, R=5, S=3, T=2}`. When this data is then passed into `FastDateParser::toArray(...)` method it fails with the `java.lang.ArrayIndexOutOfBoundsException` exception, because indexes in the map start from 1 and not 0.


### Crash #2

```java
org.apache.commons.math.MathRuntimeException$6: map has been modified while iterating
    at org.apache.commons.math.MathRuntimeException.createConcurrentModificationException(MathRuntimeException.java:373)
    at org.apache.commons.math.util.OpenIntToDoubleHashMap$Iterator.advance(OpenIntToDoubleHashMap.java:564)
    at org.apache.commons.math.linear.OpenMapRealVector.ebeMultiply(OpenMapRealVector.java:372)
    at org.apache.commons.math.linear.OpenMapRealVector.ebeMultiply(OpenMapRealVector.java:33)
```

#### Solution:

My JUnit-4.10 test that reproduces the same exception and the same stack trace:

```java
@Test(expected = ConcurrentModificationException.class)
public void test() throws InterruptedException {
    int iterations = 100000;
    OpenMapRealVector vec = new OpenMapRealVector(3);

    // just a utility lambda to set values of the vector
    BiConsumer<OpenMapRealVector, Double> setEntriesOfVector = (vector, value) -> {
        vector.setEntry(0, value);
        vector.setEntry(1, value);
        vector.setEntry(2, value);
    };

    setEntriesOfVector.accept(vec, 1.0);
    
    // we need to generate concurrent modification exception,
    // for that we modify `vec` variable from thread `t` and the main thread
    Thread t = new Thread(() -> {
        // force undelying data structure inside `OpenMapRealVector` class to update modifications count variable
        for (int i = 0; i < iterations; ++i) {
            if ((i & 1) == 0) {
                setEntriesOfVector.accept(vec, (double)i);
            }
            else {
                setEntriesOfVector.accept(vec, 0.0); // this line forces to clear the vector values and always update the modifications count variable
            }
        }
    });

    t.start();

    try {
        OpenMapRealVector vec2 = new OpenMapRealVector(3);
        setEntriesOfVector.accept(vec2, 2.0);

        for (int i = 0; i < iterations; ++i) {
            // there is `OpenIntToDoubleHashMap::Iterator` class instantiation inside, then method starts to iterate over it, calling `OpenIntToDoubleHashMap::Iterator::advance()`
            vec.ebeMultiply(vec2);
        }
    }
    catch(Exception err) {
        err.printStackTrace();
        throw err;
    }

    t.join();
}
```

#### Explanation:
1. There is `OpenIntToDoubleHashMap` data structure used inside `OpenMapRealVector` class for storing and handling vector values.
2. `OpenIntToDoubleHashMap::ebeMultiply(...)` method in its body creates an instance of `OpenIntToDoubleHashMap::Iterator` class and iterates over it. When advancing to the next element of the hashmap the check for simultaneously update happens. It is checked via modifications count variable that is stored inside `OpenIntToDoubleHashMap` class.
3. So we are able to generate the required exception if we modify `OpenIntToDoubleHashMap` data structure simultaneously with iterating over its iterator instance. My code creates a separate thread for that and performs multiple vector updates using both `OpenMapRealVector::setEntry(...)` and `OpenMapRealVector::ebeMultiply()` methods. Thus, the correct exception is thrown.

I want to notice that it is unlikely that this exception **will not** be thrown in my test, but it is still possible. For example, it is possible in several cases:

1. When thread `t` finishes its work before main thread starts its for-loop.
2. When these conditions hold at once:
    - operating system will decide to run both threads (main and `t`) on the same CPU core switching between them (e.g. if we run the test on the machine with single CPU core)
    - if `vec.ebeMultiply(...)` method never intersects with `OpenIntToDoubleHashMap::Iterator::advance()` method that is called inside `vec.setEntry(...)`.

These are some cases, but there might be more. So in order to catch this exception we should make significant number of `iterations`, in such case the chances of missing this exception will be much lower.


### About ChatGPT:

I used GPT-3.5 version in order to get some insights of how these tests might be implemented. ChatGPT was a bit userful in the 1st crash reproduction, because it gave me a good insight that the problem might be in Japanese Imperial Locale, but it still wasn't able to generate the working test code for it.

In the 2nd crash reproduction it wasn't much help from ChatGPT.
