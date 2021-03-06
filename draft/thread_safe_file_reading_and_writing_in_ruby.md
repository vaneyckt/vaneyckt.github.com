+++
date = "2014-08-17T19:23:47+00:00"
title = "Thread-safe file reading and writing in Ruby"
type = "post"
ogtype = "article"
topics = [ "ruby" ]
+++

<p>I found myself wondering about safe ways for multiple threads to write to a given file in Ruby and ended up stumbling across a veritable treasure trove of information. This post is a summary of what I learned after reading these <a href="http://blog.douglasfshearer.com/post/17547062422/threadsafe-file-consistency-in-ruby">two</a> <a href="http://www.jstorimer.com/blogs/workingwithcode/7982047-is-lock-free-logging-safe">articles</a> and a lot of googling.</p>

<h3 id="atomic-writes:af4a18686c1adbc4b137bf01aa62a2cd">Atomic Writes</h3>

<p>present enemy , identify obstcles and how to overcome them</p>

<p><a href="http://apidock.com/rails/File/atomic_write/class">Atomic writes</a> ensure that a process (or thread) that reads from a file will never see a half-written file no matter how frequent the file is being written to. The approach take here is to redirect any writes into a temporary file; once the write has finished the original file is replaced with the temporary one by making a <code>mv</code> system call. Since the <code>mv</code> system call is <a href="http://www.linuxmisc.com/9-unix-programmer/457187f6a27d0540.htm">guaranteed to be atomic</a> within <a href="http://superuser.com/questions/586540/where-does-boundary-of-file-system-lie-in-linux">file boundaries</a>, the replacement of the file will appear to be instantaneous to any reading processes.</p>

<h3 id="file-locks:af4a18686c1adbc4b137bf01aa62a2cd">File Locks</h3>

<p>Note that the above approach only guarantees that no half-written files will be read from. It does not make any guarantees about what happens when multiple processes try to write to a file at the same time. The problem in this case is two-fold. In order to write to a file a process first needs to locate where in the file to write to. This is done using the <code>lseek</code> system call. The actual writing is then done by a separate <code>write</code> system call. You can invoke this low-level call with <a href="http://ruby-doc.org/core-2.1.2/IO.html#method-i-syswrite">syswrite</a> in Ruby.</p>

<p>The first problem is that multiple processes writing to a single file can interleave their calls like so:</p>

<ul>
<li>process A performs <code>lseek</code></li>
<li>process B performs <code>lseek</code></li>
<li>process A calls <code>write</code></li>
<li>process B calls <code>write</code> and ends up overwriting the changes made by process A</li>
</ul>

<p>The second problem is the <a href="http://stackoverflow.com/questions/14387104/atomic-writes-in-linux">lack of any guarantee</a> that all the changes made by a single process will be performed in a single <code>write</code> call, e.g. the OS could interrupt an ongoing <code>write</code> call halfway through and expect you to schedule another <code>write</code> call to write the remaining data. This means that multiple processes writing to a single file could have their <code>write</code> calls interleaved, causing the new contents of the file to be out of order.</p>

<p>The solution here is to use a <a href="http://unix.stackexchange.com/questions/107038/obtain-exclusive-read-write-lock-on-a-file-for-atomic-updates">file lock</a>. There is some great Ruby code for this on the site I linked at the top of this post, as well as some examples in the <a href="http://www.ruby-doc.org/core-2.1.1/File.html#method-i-flock">docs</a>, and of course some <a href="http://stackoverflow.com/a/15304835/1420382">great posts on StackOverflow</a>.</p>

<p>In a previous post on this topic we went over the importance of using file locks when having multiple concurrent processes writing to the same file. However, using a file lock can cause a certain amount of contention between processes. This is why you&rsquo;ll sometimes see <a href="http://www.jstorimer.com/blogs/workingwithcode/7982047-is-lock-free-logging-safe">lock-free logging implementations</a>.</p>

<h2 id="lock-free-logging:af4a18686c1adbc4b137bf01aa62a2cd">Lock-free Logging</h2>

<p>Lock-free logging is all about the use of the <code>O_APPEND</code> flag. By opening a file with this flag you are effectively telling the OS to force each <code>write</code> call to this file to append its data to the end of the file. Unlike the scenario in our previous post where <code>lseek</code> and <code>write</code> calls of different processes could end up interleaved, the <code>O_APPEND</code> flag guarantees that <a href="http://pubs.opengroup.org/onlinepubs/009695399/functions/pwrite.html">&ldquo;the file offset shall be set to the end of the file prior to each write and no intervening file modification operation shall occur between changing the file offset and the write operation&rdquo;</a>. This guarantee causes lots of people to think that having multiple processes writing to the same file opened with the <code>O_APPEND</code> flag is safe to do. Unfortunately this is <a href="https://github.com/steveklabnik/mono_logger/issues/2">not correct</a>.</p>

<p>The problem boils down to an issue raised in the previous post on this topic: the <a href="http://stackoverflow.com/questions/14387104/atomic-writes-in-linux">lack of any guarantee</a> that all changes made by a single process will be performed in a single <code>write</code> call. Since <code>write</code> calls are <a href="http://stackoverflow.com/questions/7236475/what-happens-if-a-write-system-call-is-called-on-same-file-by-2-different-proces">not atomic</a>, any data written with this <code>O_APPEND</code> approach can still be reordered due to interleaved <code>write</code> operations. As an aside, do note that using <code>O_APPEND</code> will prevent multiple processes from overwriting each other&rsquo;s data.</p>

<p>So when is it safe to use this <code>O_APPEND</code> approach? In theory it should never be used; however in practice it will work correctly the vast majority of the time. <a href="http://www.jstorimer.com/">Jesse Storimer</a> asked around why this is, and got <a href="http://librelist.com/browser//usp.ruby/2013/6/5/o-append-atomicity/#c794fe8b22b5e09de4c38e6994b4e201">some answers</a>. It turns out most file systems don&rsquo;t like to interrupt <code>write</code> operations that only want to add a small amount of data, e.g. a line in a log file. This means that the majority of such <code>write</code> calls can be treated as atomic for most of the time even though it is not safe to do so. This behavior also explains the large amount of confusing information on the internet regarding this topic.</p>

    </div>
  </div>

<script data-no-instant>document.write('<script src="http://'
        + (location.host || 'localhost').split(':')[0]
		+ ':1313/livereload.js?mindelay=10"></'
        + 'script>')</script></body>
</html>
