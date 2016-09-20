---
layout: post
title: Better Developing Through Cookies
categories:
- blog
tags:
- Web Development
- Cookies
- Testing
requires_js_math: false
---

This post will look at three different ways my current project has leveraged cookies to aid in development and testing of our web application. Much credit goes to my co-worker [Sasha](https://arktronic.com) for the original impetus and implementation of one of these facets.

---

## But why male models?

So what are the advantages and disadvantages of using cookies as part of your development, debugging, and testing and how can you counteract the downsides?

On the plus side, they're easy to interact with both on the client and server side in any modern web framework. Interfaces for them can be placed in the site itself or merely exist as user scripts or javascript bookmarklets. They're automatically included with requests sent from your site to the server so you don't need to manually ensure data insertion in all places that call the server. This gives you access to enable debugging features in environments you may not otherwise have access to, including production.

On the downside, obscurity is not security, anything you can control with a cookie setting you should expect your users to be able to control as well and it shouldn't compromise your site. A co-worker related a tale of a site that determined if you were an administrator by the _lack_ of a user cookie, don't be that site. These settings can be out of sight and out of mind, so make it easy to verify their current state and consider reasonable expirations on the cookies to prevent them from hanging around and causing unexpected confusion later. Be mindful that depending on your architecture and app behavior you may need to refresh the page you are on after updating the cookie for the changes to propagate. This warrants being mindful of in a testing context especially in Single Page Apps (SPAs) where refreshing may change your context or deep linking is problematic.

### Checking a cookie in Java Spring

{% highlight java %}
public Boolean isSomeCookieSet() {
    ServletRequestAttributes sra = RequestContextHolder.getRequestAttributes();
    HttpServletRequest req = sra.getRequest();
    Cookie[] cookies = req.getCookies();
    if (cookies == null) {
        return false;
    }
    return Arrays
        .stream(cookies)
        .filter(cookie ->
          cookie.getName().equals("someKey")
          && cookie.getValue().equals("true"))
        .findFirst()
        .isPresent();
}
{% endhighlight %}

## Test Language

All of our user visible strings live in resource bundles which are sourced to our translators (Expect a blog post on the tooling we made to support this!). Or at least that's the goal, but there's been cases of strings slipping through untranslated and it's an inconvenient game of whack-a-mole verifying strings in the site against the backing resource files to make sure it's present. Even switching your browser language won't always help since some very short strings or single words often translate to the same word in many languages. This lead to the creation of our first helper, a button in our site's debug bookmarklet that would toggle on a test language mode. Any string that was going through our global message source would be modified based on success or failure. This makes it instantly visually obvious if the string was...

* Never translated
* Attempted translation but didn't exist
* Translated succesfully
* Sent through translations more than once

### Inside a Spring MessageSource
{% highlight java %}
public String getSomeTranslation(code)
    ...

    if (isTestLanguageActive()) {
        return processString(code, resolved);
    }
    return resolved;
}

private String processString(String originalCode, String translated) {
    if (translated == null) {
        return "⚠" + originalCode + "☔";
    } else {
        return "✌" + StringUtils.swapCase(translated) + "☀";
    }
}
{% endhighlight %}

![Test Language Sample](/assets/test-language.png)

This feature has been widely lauded and makes it super easy on anyone doing testing to verify the translation state of the application in any environment.

## Logging lower services

We don't actually do much work with the data in web service. Almost all of our data is sourced from a centralized API that serves many other user applications the client develops in parallel with ours. This makes tracing down data discrepancies a long and sometimes painful process. It's further complicated by the default logging through the data pipeline not always being the greatest. Lately however much of the pipeline has been retrofitted with checkpoint logging that records the entire life cycle of a request as long as a specific header is sent along with the initial request. We now needed to find the best way to know when to include this logging header from requests sent out of Spring application. By combining a Spring interceptor on outbound requests and our cookie checking logic it became simple to just navigate to the action you wish to log, turn on logging, perform the action, and disable the logging cookie.


## Permissions in testing

Our site has a local deployment option we call a Stub layer. Using Spring's dependency injection our entire data layer is swapped out with an in memory layer that provides fake data and basic persistence of changes. This is the data that our acceptance tests operate on in the various stages of our pipeline. This testing and stub layer also bypasses the authorization logging in behavior which is outside of our control in the local environment. The downside to this set up is that all of our data is basically static unless we can edit it through the site, and we can't test site behavior for various classes of users or users without certain permissions. This is a fairly serious drawback as you might guess, meaning that many things can only be tested manually and as our site begins to use more permissioning the problem was only going to grow.

Luckily cookies came to the rescue! Since our stub layer only ever exists in the local deploy we didn't need to feel bad about controlling permissions and user state in the stub objects via cookies, but what does that look like in our acceptance tests?

    Feature: Mailing List Management

        Background:
            Given User has ManageMailingLists enabled
            And Permissions are set

        Scenario: Navigate to the Mailing List page
            Given the site is successfully loaded
            When user clicks on the Mailing List button
            Then the Mailing List view is displayed
            And the edit Mailing Lists functionality is enabled

The background field is used at the top of our feature and runs before every scenario in the file, except we don't want to set permissions and refresh the browser before every single scenario, just doing it once seems far more reasonable. Let's look at the steps file that interprets the permissions.


### PermissionSteps.java

{% highlight java %}
public class PermissionSteps {
    private static Map<String, Boolean> permissionSet = new HashMap<>();
    private static Map<String, Boolean> lastPermissionSet = new HashMap<>();

    @Given("^User has (\\w+) (enabled|disabled)$")
    public void setPermissionIntoSet(String permissionKey, String valueString) {
        permissionSet.put(permissionKey, valueString.equals("enabled"));
    }

    @And("^Permissions are set$")
    public void permissionsAreSet() {
        if (!permissionSet.equals(lastPermissionSet)) {
            refreshPermissions();

            lastPermissionSet.clear();
            lastPermissionSet.putAll(permissionSet);
            permissionSet.clear();
        }
    }

    private void refreshPermissions() {
        lastPermissionSet.keySet()
            .stream()
            .forEach(p -> BrowserDriver.getCurrentDriver()
              .manage().deleteCookieNamed(p));

        permissionSet.entrySet()
            .stream()
            .forEach(p ->
                BrowserDriver.getCurrentDriver()
                  .manage().addCookie(
                    new Cookie(p.getKey(), p.getValue().toString())
                ));
        BrowserDriver.getCurrentDriver().navigate().refresh();
    }
}
{% endhighlight %}

These steps were designed to minimize the number of times we refresh the browser (which is slowwww). By persisting the previous and current permissions we can wait until all permissions are set for the test file, and then only actually update them if any permissions have changed between the permission sets. This means we only refresh the browser once when entering a file and only if that file changes the permissions from the previous test run. Hopefully these additions will allow more flexible tests and make it easier to assert conditions outside of the single path we've been able to test up until now.


Cookies can be an incredibly convenient tool for enabling rich debugging and testing in your web application, and can simplify some common problems you might run into.
