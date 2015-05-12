## SNRSearchIndex: Core Data search backed by SearchKit

`SNRSearchIndex` is a simple wrapper for the [SearchKit framework](http://developer.apple.com/library/mac/#documentation/UserExperience/Reference/SearchKit/Reference/reference.html) that is specifically focused around providing lightning fast search for Core Data databases. SearchKit is the framework that Apple's Spotlight is built on, and it's primary use is full text search of documents. That said, it's so fast that it's applicable to a wide variety of cases. I tried using SearchKit with Core Data on a hunch, and the results were far better than I expected. It's really not the code itself that's the valuable part of this project, but the concept of using a FTS framework for searching a Core Data DB.

**SNRSearchIndex must be compiled under ARC.** If your project has not been converted to use ARC yet, you can add the `-f-objc-arc` flag to `SNRSearchIndex.m` in the Compile Sources build phase for your project. 

## Usage

### Creating/Opening a Search Index

This is fairly self explanatory. The `-initByOpeningSearchIndexAtURL:persistentStoreCoordinator` and `-initByCreatingSearchIndexAtURL:persistentStoreCoordinator` methods of `SNRSearchIndex` will open or create a search index file at the specified URL. If no URL is specified, it will automatically set the URL as **[directory that the first NSPersistentStore is located in]/searchindex**. 

    SNRSearchIndex *index = [[SNRSearchIndex alloc] initByCreatingSearchIndexAtURL:nil persistentStoreCoordinator:self.persistentStoreCoordinator];

### Using a Shared Search Index

Often times, your application will only use a single search index, and it's inconvenient having to maintain references to a search index from all of the various objects that use it. Therefore, `SNRSearchIndex` has the `+sharedIndex` and `+setSharedIndex:` class methods to get and set a shared index that will be easily accessible from anywhere in your code.

    // Set the shared index when you first create it
    [SNRSearchIndex setSharedIndex:index]
    
    // From another class...
    SNRSearchIndex *theIndex = [SNRSearchIndex sharedIndex];

### Setting Keywords for Managed Objects

Setting keywords to save in the search index for your managed objects is simple. Create an `NSArray` of `NSString`s of the keywords for that object and call `-setKeywords:forObjectID:`.  The reason that `NSManagedObjectID`'s are used vs. the `NSManagedObject`s themselves is because `SearchKit`, and therefore `SNRSearchIndex`, is threadsafe and passing around the managed objects themselves is not thread safe because they are confined to a single managed object context.

    Car *car = …; // NSManagedObject subclass
    NSArray *keywords = [NSArray arrayWithObjects:@"car", @"red", @"leather", @"v6", nil];
    [[SNRSearchIndex sharedIndex] setKeywords:keywords forObjectID:[car objectID]];
    
One of the main annoyances with using a separate search index is that you need to keep it in sync with the managed object context when an attribute of a model object changes that affects the keywords in the search index. Therefore, it would be convenient to use custom accessors or KVO in your `NSManagedObject` subclass to automatically update the keywords in the search index. Example:

    @interface Game : NSManagedObject
    @property (nonatomic, retain) NSString *name;
    @end
    
    @interface Game (CoreDataGeneratedPrimitiveAccessors)
    - (void)setPrimitiveName:(NSString*)primitiveName;
    @end
    
    @implementation Game
    - (void)setName:(NSString*)value
    {
    	[self willChangeValueForKey:@"name"];
    	[self setPrimitiveName:value];
    	[self didChangeValueForKey:@"name"];
    	NSArray *keywords = [value componentsSeparatedByString:@" "]; // Create keywords by separating the name by its spaces
    	[[SNRSearchIndex sharedIndex] setKeywords:keywords forObjectID:[self objectID]];
    }
    @end

### Synchronizing Saves with NSManagedObjectContext

Calling `-flush` on an `SNRSearchIndex` will commit any changes to the backing store, much like calling `-save:` on an `NSManagedObjectContext`. Instead of having to call save on both every time changes are made, it is far simpler to implement a method in a category for `NSManagedObjectContext` that saves both the search index and the MOC:

    @interface NSManagedObjectContext (SearchIndex)
    - (BOOL)saveContextAndSearchIndex:(NSError**)error;
    @end
    
    @implementation NSManagedObjectContext (SearchIndex)
    - (BOOL)saveContextAndSearchIndex:(NSError**)error
    {
    	BOOL success = [[SNRSearchIndex sharedIndex] flush];
    	if (![self save:error]) {
    		return NO;
    	}
    	return success;
    }
    @end

### Creating a Search Query

The `SNRSearchIndexSearch` object encapsulates a single search query. Use the `-searchForQuery:options:` method of `SNRSearchIndex` to create a search query object. The values for the `options` parameter are documented in the header. `SearchKit` supports a specific set of query operators that are described [here](http://developer.apple.com/library/mac/documentation/UserExperience/Reference/SearchKit/Reference/reference.html#//apple_ref/c/func/SKSearchCreate). 

    // Substring search for 'berry'
    
    SNRSearchIndexSearch *search = [[SNRSearchIndex sharedIndex] searchForQuery:@"*berry" options:0];  
    
    // this would return results for 'strawberry', 'blackberry', etc.
    
### Performing a Search Query

Search queries are performed using the `-findMatchesWithFetchLimit:maximumTime:handler` method of `SNRSearchIndexSearch`. These parameters are documented in the header. This method can be called from a background thread, and once the search results are retrieved, they are passed to the handler block as an array of `SNRSearchIndexSearchResult` objects. Mapping these results to actual `NSManagedObject`'s is trivial:

    [search findMatchesWithFetchLimit:10 maximumTime:1.0 handler:^(NSArray *results) {
    	NSMutableArray *objects = [NSMutableArray arrayWithCapacity:[results count]];
    	for (SNRSearchIndexSearchResult *result in results) {
    		NSManagedObject *object = [self.managedObjectContext existingObjectWithID:result.objectID error:nil];
    		if (object) { [objects addObject:object]; }
    	}
    	// Do something with the objects array
    }];

**Note:** If you're implementing a "type to search" feature, it would be a good idea to cancel the previous search query before executing a new one. Implement a property setter that retains a reference to the current search query and cancels the old one when a new one is set:

    - (void)setCurrentSearch:(SNRSearchIndexSearch*)search
    {
    	if (_currentSearch != search) {
    		[_currentSearch cancel];
    		_currentSearch = search;
    	}
    }

### Relevance Scores

`SearchKit` has support for relevance scores, which rank search results based on some algorithms that determine their similarity to the search query. `SNRSearchIndex` exposes these relevance scores, but they probably aren't that accurate nor useful for this purpose (they're more useful for full text search).

## About Me

I'm Indragie Karunaratne, a 17 year old Mac and iOS developer. Visit my [website](http://indragie.com) to check out my portfolio and get in touch with me.

## Licensing

`SNRSearchIndex` is licensed under the [BSD license](http://www.opensource.org/licenses/bsd-license.php).
