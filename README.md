
<img style="float: right;" src="https://raw.githubusercontent.com/sushihangover/RealmThread/master/media/SushiHangover.RealmThread.png">

##SushiHangover.RealmThread

**An Action/Task Message Pump for running commands on a dedicated Realm thread.**

[![Join the chat at https://gitter.im/RealmThread/Lobby](https://badges.gitter.im/RealmThread/Lobby.svg)](https://gitter.im/RealmThread/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge) [![Build status](https://ci.appveyor.com/api/projects/status/dtbnjnda7h115bjv/branch/master?svg=true)](https://ci.appveyor.com/project/sushihangover/realmthread/branch/master)

## GitHub Repo

[https://github.com/sushihangover/RealmThread](https://github.com/sushihangover/RealmThread)

##API Documention

[https://sushihangover.github.io/RealmThread/
](https://sushihangover.github.io/RealmThread/)

##Nuget:

<div class="nuget-badge">
<p>
<code>
Install-Package SushiHangover.RealmThread
</code>
</p>
</div>
        
**Nuget.org:** [https://www.nuget.org/packages/sushihangover.realmthread/](https://www.nuget.org/packages/sushihangover.realmthread/)

##Issues?

Post an [Issue](https://github.com/sushihangover/RealmThread/issues) on Github

##Usage:

##RealmThread:

Instance a new `RealmThread` by passing a `RealmConfiguration`, either from an existing `Realm` instance (`Realm.Config`) or constructing one yourself.

	var realmConfig = new RealmConfiguration("myRealm.db");
	var realmThread = new RealmThread(realmConfig);

##IDisposable:
`RealmThread` implements `IDisposable` and to ensure data integrity call `Dispose` to ensure that all queued work on the dedicated Realm thread is completed.

	using (var realmThread = new RealmThread(realmConfig))
	{
		// Perform some work with the RealmThread
	}

**Or:**

	var realmThread = new RealmThread(realmConfig);
	// At some future point:
	realmThread.Dispose();

##`InvokeAsync`: Tasks and Continuations

To ensure that `Task` *continuations* are performed on the same thread, use `InvokeAsync` when your Task contains other awaited Tasks, otherwise use `Invoke` or `BeginInvoke` to execute an `Action`.

**Note:** This is a blocking call and will not return till all the queued items up till, including this one along with its continuations are executed.

	await realmThread.InvokeAsync(async r =>
	{
		await Task.FromResult(true); // Simulate some Task, i.e. a httpclient request.... 
		// The following continuations will be executed on the proper thread
		r.Write(() =>
		{
			var obj = r.CreateObject<KeyValueRecord>();
			obj.Key = "key";
		});
	});

##`BeginInvoke`: A Non-blocking Action:

`BeginInvoke` will queue an `Action` and immediately return (`Fire & Forget`). 

**Note:** The items in the work queue are executed in the order they were queued (FIFO) and to mantain data integrity they can not be reordered once added.
	   
	realmThread.BeginInvoke(internalRealmOnThread =>
	{
		internalRealmOnThread.Write(() =>
		{
			var obj = internalRealmOnThread.CreateObject<KeyValueRecord>();
			obj.Key = "key";
		});
	});

##`Invoke`: A Blocking Action:

`Invoke` will queue an `Action` and wait till it is executed.

**Note:** The items in the work queue are executed in the order they were added, so until all the queued items up till and including this one are executed, this call will block.

	realmThread.Invoke(internalRealmOnThread =>
	{
		internalRealmOnThread.Write(() =>
		{
			var obj = internalRealmOnThread.CreateObject<KeyValueRecord>();
			obj.Key = "key";
		});
	});

##Transaction Helpers


`RealmThread` has a built-in `Realms.Transaction` and a matching set of APIs that can be used instead of using a `Realms.Transaction` via a *captured* variable:

* Methods:
 * `BeginTransaction`
 * `CommitTransaction`
 * `RollbackTransaction`
* Properties:
 * `InTransaction`

####Example:
		
	realmThread.BeginTransaction();
	realmThread.Invoke(r =>
	{
		var obj = r.CreateObject<KeyValueRecord>();
		obj.Key = "key";
	});
	if (realmThread.InTransaction)
	{
	    realmThread.CommitTransaction();
	}

####**NOTE:** When a `RealmThread` object is disposed, if the built-in transaction is open, it will perform a **`Rollback`**. If you wish to override this behavoir, when creating a `RealmThread`, set the `autoCommit` parameter in the constructor to `true`.


##Captured variables

You can use captured variables within your `RealmThread`-based `Action` and `Task` to *pass* variables between the calling thread and the internal thread that `RealmThread` maintains a `Realm` instance on.

You can not pass a **managed** `RealmObject` or `RealmResult` in this manner as its access has to be made on the same thread it was instanced. 

If needed, you could copy a **managed** `RealmObject` to a non-managed `RealmObject` and use this non-managed `RealmObject` variable as your captured variable that is passed between threads.

###Non-managed RealmObject amoung threads:

	var keyValueRecord = new KeyValueRecord();
	realmThread.Invoke(r =>
	{
		var obj = r.ObjectForPrimaryKey<KeyValueRecord>(keyToFind);
		keyValueRecord.Key = obj.Key;
		keyValueRecord.Value = obj.Value;
	});
	Console.WriteLine($"{keyValueRecord.Key}:{keyValueRecord.Value}");

###Example of passing a Transaction:

**Note:** This was the primary way to control a `Transaction` over multiple invokes before the Transaction helper api was added.

	Transaction trans = null;
	realmThread.Invoke(r =>
	{
		trans = r.BeginWrite();
	});
	// Start a bunch of RealmObject inserts/edits
	realmThread.Invoke(r =>
	{
		var obj = r.CreateObject<KeyValueRecord>();
		obj.Key = "key";
	});
	// Complete the transaction
	t.Invoke(r =>
	{
		if (trans != null)
			trans.Commit();
	});

##Parallel Write / Read Example:

While this is a contrived example, it shows one `RealmThread` being used to only add records and another being used to read these new records.

**Note:** The `toWrite` variable is a pre-populated `Dictionary<string, byte[]>`, see the performace tests in the source repo, within the `RealmThread.Tests.Shared` project, for creation details...

	using (var blockingQueue = new BlockingCollection<string>())
	using (var realmThreadWrite = new RealmThread(aRealmInstance.Config))
	using (var realmThreadRead = new RealmThread(aRealmInstance.Config))
	{
		Parallel.Invoke(() =>
		{
			realmThreadWrite.Invoke(threadSafeWriteRealm =>
			{
				foreach (var kvp in toWrite)
				{
					// Individual record write transactions so the other RealmThread can read asap
					threadSafeWriteRealm.Write(() =>
					{
						var obj = threadSafeWriteRealm.CreateObject(typeof(KeyValueRecord).Name);
						obj.Key = kvp.Key;
						obj.Value = kvp.Value;
					});
					blockingQueue.Add(kvp.Key);
				}
				blockingQueue.CompleteAdding();
			});
		},
		() =>
		{
			realmThreadRead.Invoke((threadSafeReadRealm) =>
			{
				foreach (var key in blockingQueue.GetConsumingEnumerable())
				{
					// Refresh() is automatically called at the beginning of each BeginInvoke/Invoke, 
					// so if we are within the RealmPump block and need to see the latest changes 
					// from other Realm instances, call Refresh manually
					threadSafeReadRealm.Refresh();
					var record = threadSafeReadRealm.ObjectForPrimaryKey<KeyValueRecord>(key);
					Assert.NotNull(record);
					Assert.Equal(key, record.Key);
				}
			});
		});
	}



##Build from Source:

From the cmd line using the amazing [cake](http://cakebuild.net):
<a href="http://cakebuild.net">
<img src="http://cakebuild.net/Content/img/logo.png" alt="cake"/>
</a>
<div class="code">
./build.sh -t Build
</div>

##Build Documention:

API Reference documention is built via the great <a href="http://www.doxygen.org/index.html">
<img src="http://www.stack.nl/~dimitri/doxygen/doxygen.png" alt="doxygen"/>
</a>

<div class="code">
doxygen Doxygen/realmthread.config
</div>

##Build Nuget Package:

<div class="code">
./build.sh -t Package
</div>

##Publish Nuget:

<pre>
<div class="code">export NUGET_APIKEY={APIKEY}
export GITHUB_TOKEN={TOKEN/PASSWORD}
export GITHUB_USERNAME={EMAILADDRESS}
export NUGET_SOURCE=https://www.nuget.org/api/v2/package
./build.sh -t PublishPackages
</div>
</pre>

<center><sub>Thread Icon within the RealmThread Logo:</sub><br/>
<sub>
Icons made by <a href="http://www.freepik.com" title="Freepik">Freepik</a> from <a href="http://www.flaticon.com" title="Flaticon">www.flaticon.com</a> is licensed by <a href="http://creativecommons.org/licenses/by/3.0/" title="Creative Commons BY 3.0" target="_blank">CC 3.0 BY</a>
</sub></center>

<head>
<style>
.nuget-badge code {
    -moz-border-radius: 5px;
    -webkit-border-radius: 5px;
    background-color: #202020;
    border: 4px solid silver;
    border-radius: 5px;
    box-shadow: 2px 2px 3px #6e6e6e;
    color: #e2e2e2;
    display: block;
    font: 1.0em 'andale mono', 'lucida console', monospace;
    line-height: 1.5em;
    overflow: auto;
    padding: 15px
}
.nuget-badge code::before {
    content: "PM> "
}
.code {
    -moz-border-radius: 5px;
    -webkit-border-radius: 5px;
    background-color: #202020;
    border: 4px solid silver;
    border-radius: 5px;
    box-shadow: 2px 2px 3px #6e6e6e;
    color: #e2e2e2;
    display: block;
    font: 1.0em 'andale mono', 'lucida console', monospace;
    line-height: 1.5em;
    overflow: auto;
    padding: 15px
}

</style>
</head>
