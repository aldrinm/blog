---
title: JUnit Jupiter's Display name generators
date: "2024/04/18 00:00:00"
---

When writing Junit tests, it is nice to have a clear and readable test name. It may not be necessary, but it helps while scanning through a long list of test names. 

JUnit 5 has a handy feature to customize the display name of tests. 

## @DisplayName
Using `@DisplayName` you can provide your own test name. IDEs can then show a more readable name for the test.

*(Bonus - emojis and special characters can be freely used)*

{% codeblock lang:java line_number:false highlight: true %}
@Test
@DisplayName("Test that a Book is initialized correctly 🪄📖")
void test_book_initialization() {
    // Test logic here
}
{% endcodeblock %}

will be displayed in IntelliJ as,
{% asset_img display_intellij.png 'IntelliJ's @DisplayName handling of the test name' %}


## @DisplayNameGenerator
With `@DisplayName` you have to add a description to every test method.
If you don't want to do that but just want regular test cases without 
much effort, JUnit provides `@DisplayNameGenerator` to customize the name. 

For instance, there is an already available `DisplayNameGenerator.ReplaceUnderscores` that replaces the underscores in your test names.

So,
{% codeblock lang:java line_number:false highlight: true %}
@SpringBootTest
@DisplayNameGeneration(DisplayNameGenerator.ReplaceUnderscores.class)
class DisplayNameGeneratorUnderscoreTests {
    @Test
    void test_book_initialization() {
        // Test logic here
    }

    @Test
    void test_book_set_title() {
        // Test logic here
    }

    @Test
    void test_book_set_author() {
        // Test logic here
    }
}
{% endcodeblock %}

gets displayed by IntelliJ, without underscores, as,
{% asset_img underscore_tests.png 'IntelliJ's handling of ReplaceUnderscores' %}


## Customizing @DisplayNameGenerator
What if your project uses camel case for all methods including 
test methods? There isn't an out-of-the-box generator that can reformat that, 
but it is very easy to write one yourself. Extend `DisplayNameGenerator.Simple` and override the
methods you need. (Alternatively, you can also implement `org.junit.jupiter.api.DisplayNameGenerator`)

*(following code is also available at https://github.com/aldrinm/display-name-generators/blob/main/src/test/java/aldrin/displaynamegenerators/CamelCaseGenerator.java)*

{% codeblock lang:java line_number:false highlight: true %}
    import org.apache.commons.lang3.StringUtils;
    import org.junit.jupiter.api.DisplayNameGenerator;
    import java.lang.reflect.Method;
    
    public class CamelCaseGenerator extends DisplayNameGenerator.Simple {
    
        @Override
        public String generateDisplayNameForMethod(Class<?> aClass, Method method) {
            return formatCamelCaseName(method.getName());
        }
    
        private String formatCamelCaseName(String camelCaseString) {
            return StringUtils.join(StringUtils.splitByCharacterTypeCamelCase(camelCaseString), " ");
        }
    
    }
{% endcodeblock %}
Note that the above class depends on `StringUtils` from Apache Commons Lang, so add this dependency to your project if needed,

{% codeblock lang:xml line_number:false highlight: true %}
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.14.0</version>
</dependency>
{% endcodeblock %}


{% codeblock lang:java line_number:false highlight: true %}
@SpringBootTest
@DisplayNameGeneration(CamelCaseGenerator.class)
class DisplayNameGeneratorCamelTests {

    @Test
    void testBookInitialization() {
        //for demo
    }

    @Test
    void testBookSetTitle() {
        //for demo
    }

    @Test
    void testBookSetAuthor() {
        //for demo
    }
}
{% endcodeblock %}

gets displayed by IntelliJ, as,
{% asset_img camel_case.png 'IntelliJ's handling of CamelCaseGenerator' %}



## Project-Wide Configuration
Finally, you can configure a default generator for the whole project, by specifying it in properties (preferably `src/test/resources/junit-platform.properties` )

{% codeblock lang:properties line_number:false highlight: true %}
junit.jupiter.displayname.generator.default = \
    org.junit.jupiter.api.DisplayNameGenerator$ReplaceUnderscores
{% endcodeblock %}


For more details, refer to the [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/#writing-tests-display-name-generator
).