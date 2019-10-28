# dropwizard-api-key-bundle

A [Dropwizard][dropwizard] bundle that provides a simple way to manage API keys for callers of
your service. The bundle provides support for authentication only; authorization is supported
by optionally providing an `Authorizer` as documented below.

Convey's fork adds support for newer versions of Dropwizard and was further customized for use for all Convey projects, most specifically allowing this extension to be built and used with Java 12+, as well as being easily released to the Maven Central repository.

## Getting Started

Just add this maven dependency to get started:

```xml
<dependency>
  <groupId>io.dropwizard-bundles</groupId>
  <artifactId>dropwizard-api-key-bundle</artifactId>
  <version>1.0.0</version>
</dependency>
```

If you only need authentication and a default `Principal` implementation add the default
version of the bundle to your environment:

```java
public class MyApplication extends Application<MyConfiguration> {
  @Override
  public void initialize(Bootstrap<MyConfiguration> bootstrap) {
    bootstrap.addBundle(new ApiKeyBundle<>());
  }

  @Override
  public void run(MyConfiguration cfg, Environment env) throws Exception {
    // ...
  }
}
```

If you need to provide an `Authorizer` or a different `Principal` (extending type), or both,
add the bundle to your environment and provide the type extending the `Principal` interface, an
implementation of the `Authorizer` and `PrincipalFactory` as appropriate:

```java
public class MyApplication extends Application<MyConfiguration> {
  @Override
  public void initialize(Bootstrap<MyConfiguration> bootstrap) {
    bootstrap.addBundle(new PrincipalApiKeyBundle<>(User.class, new PrincipalFactory<User>() {
      @Override
      public User create(String name) {
        // Do something interesting...
        return new User(name);
      }}, new Authorizer<User>() {
        @Override
        public boolean authorize(User user, String role) {
          return user.getName().equals("application-1") && role.equals("ADMIN");
        }
    }));
  }

  @Override
  public void run(MyConfiguration cfg, Environment env) throws Exception {
    // ...
  }
}
```

Additionally you can also pass an `UnauthorizedHandler` when creating the bundle, which is useful
if you need to customize the unauthorized response (e.g. type or entity).

You will also need to make your `MyConfiguration` class implement `ApiKeyBundleConfiguration` in
order to provide the bundle with the necessary information it needs to know your API keys.

```java
public class MyConfiguration implements ApiKeyBundleConfiguration {
  @Valid
  @NotNull
  @JsonProperty("authentication")
  private final ApiKeyConfiguration apiKeyConfiguration = null;

  /**
   * Return the API key configuration.
   */
  @Override
  public ApiKeyConfiguration getApiKeyConfiguration() {
    return apiKeyConfiguration;
  }
}
```

Now you can use API key based authentication in your application by declaring a method on a resource
that has an `@Auth` annotated `Principal` parameter (or an extending type).  See the
[Dropwizard Authentication][authentication] documentation for more details.  The name of the `Principal`
will be the name of the application that made the request if the authentication process was
successful.

As far as configuration goes you can define your API keys, and roles for those applications in
your application's config file. Assuming, like in the above example, you called your API key
configuration element `authentication` then you can use a config file like this:

```yaml
authentication:
  basic-http:
    cache-spec: maximumSize=1000, expireAfterAccess=10m
    realm: MyApplication
    keys:
      application-1: api-key-1
      application-2: api-key-2
    roles:
      application-1:
        - admin
        - user
      application-2:
        - user
```

[dropwizard]: http://dropwizard.io
[authentication]: http://www.dropwizard.io/1.0.0/docs/manual/auth.html

# Releasing

## Initial setup
In order to release this POM to the Maven Central repository you'll need to do some setup
the very first time.

### Find the Convey oss.sonatype.org account
We'll be releasing to the Maven Central repository via oss.sonatype.org using the 
sonatype@getconvey.com account.

### Create a GPG signing key
One of the requirements for releasing binaries to oss.sonatype.org is that
they are signed with a GPG key.  This ensures that it's more difficult for
a hacker to change binaries should maven central or sonatype ever be
compromised. *You should use your own email address*. Instructions for how 
to create a GPG key and to publish it can be found 
[here](http://central.sonatype.org/pages/working-with-pgp-signatures.html).

### Create a $HOME/.m2/settings.xml file.
Now that you have a oss.sonatype.org account, you'll need to tell maven
your username and password (so that it can upload files).  We'll do this
using a `settings.xml` file in your `$HOME/.m2` directory.  Its contents
should look something like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/Settings/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/Settings/1.0.0
                      http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <server>
      <id>oss.sonatype.org</id>
      <username>sonatype@getconvey.com</username>
      <password>YOUR PASSWORD HERE</password>
    </server>
  </servers>
</settings>
```

Don't put these credentials or this file anywhere other than your laptop!

## Creating a new release

To release this POM to the Maven Central repository you'll need to use
the `sonatype` profile and deploy the POM as you would otherwise normally
do.

```bash
# Prepare and then perform the release.
mvn release:prepare release:perform
```

If the above command fails with "gpg: problem with the agent: Inappropriate
ioctl for device", run the following: `export GPG_TTY=$(tty)`.

Now at this point your pom should have been uploaded to oss.sonatype.org and
is waiting to be released. Log in to the Sonatype console, find the POM's bundle
in the staging repository, close it and then release it.