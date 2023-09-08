# AI-based Crash Reproduction

## Task 1:
> Why are you interested in this project and how do you see its implementation?

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

There is a difference in the exception message probably because of the platform and setup differences. I used: 
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
org.apache.commons.math.MathRuntimeException$6: the addressing table has been modified during the iteration 
    at org.apache.commons.math.MathRuntimeException.createConcurrentModificationException(MathRuntimeException.java:373)
    at org.apache.commons.math.util.OpenIntToDoubleHashMap$Iterator.advance(OpenIntToDoubleHashMap.java:564)
    at org.apache.commons.math.linear.OpenMapRealVector.ebeMultiply(OpenMapRealVector.java:372)
    at org.apache.commons.math.linear.OpenMapRealVector.ebeMultiply(OpenMapRealVector.java:33)
```

