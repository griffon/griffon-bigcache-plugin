
Bigcache support
----------------

Plugin page: [http://artifacts.griffon-framework.org/plugin/bigcache](http://artifacts.griffon-framework.org/plugin/bigcache)


The Bigcache plugin enables lightweight access to [BigCache][1].
This plugin does NOT provide domain classes nor dynamic finders like GORM does.

Usage
-----
Upon installation the plugin will generate the following artifacts in `$appdir/griffon-app/conf`:

 * BigcacheConfig.groovy - contains the cache definitions.
 * BootstrapBigcache.groovy - defines init/destroy hooks for data to be manipulated during app startup/shutdown.

A new dynamic method named `withBigcache` will be injected into all controllers,
giving you access to a `org.bigcache.BigCacheManager` object, with which you'll be able
to make calls to the cache. Remember to make all cache calls off the EDT
otherwise your application may appear unresponsive when doing long computations
inside the EDT.

This method is aware of multiple caches. If no bigcacheManagerName is specified when calling
it then the default cache will be selected. Here are two example usages, the first
queries against the default cache while the second queries a cache whose name has
been configured as 'internal'

    package sample
    class SampleController {
        def queryAllDatabases = {
            withBigcache { bcmName, bcm -> ... }
            withBigcache('internal') { bcmName, bcm -> ... }
        }
    }

This method is also accessible to any component through the singleton `griffon.plugins.bigcache.BigcacheConnector`.
You can inject these methods to non-artifacts via metaclasses. Simply grab hold of a particular metaclass and call
`BigcacheEnhancer.enhance(metaClassInstance)`.

Configuration
-------------
### Dynamic method injection

The `withBigcache()` dynamic method will be added to controllers by default. You can
change this setting by adding a configuration flag in `griffon-app/conf/Config.groovy`

    griffon.bigcache.injectInto = ['controller', 'service']

### Events

The following events will be triggered by this addon

 * BigcacheConnectStart[config, bigcacheManagerName] - triggered before connecting to the bigcacheManager
 * BigcacheConnectEnd[bigcacheManagerName, bigcacheManager] - triggered after connecting to the bigcacheManager
 * BigcacheDisconnectStart[config, bigcacheManagerName, bigcacheManager] - triggered before disconnecting from the bigcacheManager
 * BigcacheDisconnectEnd[config, bigcacheManagerName] - triggered after disconnecting from the bigcacheManager

### Multiple Stores

The config file `BigcacheConfig.groovy` defines a default manager block. As the name
implies this is the cache used by default, however you can configure named bigcache managers
by adding a new config block. For example connecting to a manager whose name is 'internal'
can be done in this way

    managers {
        internal {
            caches {
                data = [
                    capacity = '1G'
                ]
            }
        }
    }

This block can be used inside the `environments()` block in the same way as the
default manager block is used.

### Example

A trivial sample application can be found at [https://github.com/aalmiray/griffon_sample_apps/tree/master/persistence/bigcache][2]

Testing
-------
The `withBigcache()` dynamic method will not be automatically injected during unit testing, because addons are simply not initialized
for this kind of tests. However you can use `BigcacheEnhancer.enhance(metaClassInstance, bigcacheProviderInstance)` where 
`bigcacheProviderInstance` is of type `griffon.plugins.bigcache.BigcacheProvider`. The contract for this interface looks like this

    public interface BigcacheProvider {
        Object withBigcache(Closure closure);
        Object withBigcache(String bigcacheManagerName, Closure closure);
        <T> T withBigcache(CallableWithArgs<T> callable);
        <T> T withBigcache(String bigcacheManagerName, CallableWithArgs<T> callable);
    }

It's up to you define how these methods need to be implemented for your tests. For example, here's an implementation that never
fails regardless of the arguments it receives

    class MyBigcacheProvider implements BigcacheProvider {
        Object withBigcache(String bigcacheManagerName = 'default', Closure closure) { null }
        public <T> T withBigcache(String bigcacheManagerName = 'default', CallableWithArgs<T> callable) { null }
    }

This implementation may be used in the following way

    class MyServiceTests extends GriffonUnitTestCase {
        void testSmokeAndMirrors() {
            MyService service = new MyService()
            BigcacheEnhancer.enhance(service.metaClass, new MyBigcacheProvider())
            // exercise service methods
        }
    }


[1]: http://code.google.com/p/bigcache-org
[2]: https://github.com/aalmiray/griffon_sample_apps/tree/master/persistence/bigcache

