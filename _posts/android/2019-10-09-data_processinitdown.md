---
layout: post
title:  Android中Activity启动(下)
category: Android
tags: Android进程创建
---
* content
{:toc}
#### 前言
在上一篇中已经提到了，分为两步，一步是开始activity，一步是创建进程，在上面的一篇中已经把开启  
开启activity的部分讲到了，下面说下应用是怎么创建进程的.其中大量参考了gityuan先生的博客。并  
根据博客的指示，来把代码跟了一遍.
#### 1.1 涉及到的源码

```java
/frameworks/base/core/java/com/android/internal/os/
    - ZygoteInit.java
    - ZygoteConnection.java
    - RuntimeInit.java
    - Zygote.java

/frameworks/base/core/java/android/os/Process.java
/frameworks/base/core/jni/com_android_internal_os_Zygote.cpp
/frameworks/base/core/jni/AndroidRuntime.cpp
/frameworks/base/cmds/app_process/App_main.cpp （内含AppRuntime类）

/bionic/libc/bionic/fork.cpp
/bionic/libc/bionic/pthread_atfork.cpp

/libcore/dalvik/src/main/java/dalvik/system/ZygoteHooks.java
/art/runtime/native/dalvik_system_ZygoteHooks.cc
/art/runtime/Runtime.cc
/art/runtime/Thread.cc
/art/runtime/signal_catcher.cc
```


在前面说到，启动进程的时候会调用的AMS类里面的startProcessLocked方法，最终还会调用到  
Process.start这个方法

```java
public static final ProcessStartResult start(final String processClass,
480                                  final String niceName,
481                                  int uid, int gid, int[] gids,
482                                  int runtimeFlags, int mountExternal,
483                                  int targetSdkVersion,
484                                  String seInfo,
485                                  String abi,
486                                  String instructionSet,
487                                  String appDataDir,
488                                  String invokeWith,
489                                  String[] zygoteArgs) {
490        return zygoteProcess.start(processClass, niceName, uid, gid, gids,
491                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
492                    abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
493    }
```

接着调用ZygoteProcess.java里面的startViaZygote方法

```java
private Process.ProcessStartResult startViaZygote(final String processClass,
358                                                      final String niceName,
359                                                      final int uid, final int gid,
360                                                      final int[] gids,
361                                                      int runtimeFlags, int mountExternal,
362                                                      int targetSdkVersion,
363                                                      String seInfo,
364                                                      String abi,
365                                                      String instructionSet,
366                                                      String appDataDir,
367                                                      String invokeWith,
368                                                      boolean startChildZygote,
369                                                      String[] extraArgs)
370                                                      throws ZygoteStartFailedEx {
371    .........................................................................
436
437        synchronized(mLock) {
438            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
439        }
440    }
```

向在zygote进程里面注册的socket发送消息，在zygote进程里面注册新的进程.
```java
private static Process.ProcessStartResult zygoteSendArgsAndGetResult(
281            ZygoteState zygoteState, ArrayList<String> args)
282            throws ZygoteStartFailedEx {
283        try {
284            // Throw early if any of the arguments are malformed. This means we can
285            // avoid writing a partial response to the zygote.
286            int sz = args.size();
287            for (int i = 0; i < sz; i++) {
288                if (args.get(i).indexOf('\n') >= 0) {
289                    throw new ZygoteStartFailedEx("embedded newlines not allowed");
290                }
291            }
292
293            /**
294             * See com.android.internal.os.SystemZygoteInit.readArgumentList()
295             * Presently the wire format to the zygote process is:
296             * a) a count of arguments (argc, in essence)
297             * b) a number of newline-separated argument strings equal to count
298             *
299             * After the zygote process reads these it will write the pid of
300             * the child or -1 on failure, followed by boolean to
301             * indicate whether a wrapper process was used.
302             */
303            final BufferedWriter writer = zygoteState.writer;
               //拿到输入流，写入服务器
304            final DataInputStream inputStream = zygoteState.inputStream;
305
306            writer.write(Integer.toString(args.size()));
307            writer.newLine();
308
309            for (int i = 0; i < sz; i++) {
310                String arg = args.get(i);
311                writer.write(arg);
312                writer.newLine();
313            }
314
315            writer.flush();
316
317            // Should there be a timeout on this?
318            Process.ProcessStartResult result = new Process.ProcessStartResult();
319
320            // Always read the entire result from the input stream to avoid leaving
321            // bytes in the stream for future process starts to accidentally stumble
322            // upon.
              //等待返回的pid
323            result.pid = inputStream.readInt();
324            result.usingWrapper = inputStream.readBoolean();
325
326            if (result.pid < 0) {
327                throw new ZygoteStartFailedEx("fork() failed");
328            }
329            return result;
330        } catch (IOException ex) {
331            zygoteState.close();
332            throw new ZygoteStartFailedEx(ex);
333        }
334    }
```

去链接远程的socket
```java
@GuardedBy("mLock")
563    private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
564        Preconditions.checkState(Thread.holdsLock(mLock), "ZygoteProcess lock not held");
565
566        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
567            try {
568                primaryZygoteState = ZygoteState.connect(mSocket);
569            } catch (IOException ioe) {
570                throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
571            }
572            maybeSetApiBlacklistExemptions(primaryZygoteState, false);
573            maybeSetHiddenApiAccessLogSampleRate(primaryZygoteState);
574        }
575        if (primaryZygoteState.matches(abi)) {
576            return primaryZygoteState;
577        }
578
579        // The primary zygote didn't match. Try the secondary.
580        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
581            try {
582                secondaryZygoteState = ZygoteState.connect(mSecondarySocket);
583            } catch (IOException ioe) {
584                throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
585            }
586            maybeSetApiBlacklistExemptions(secondaryZygoteState, false);
587            maybeSetHiddenApiAccessLogSampleRate(secondaryZygoteState);
588        }
589
590        if (secondaryZygoteState.matches(abi)) {
591            return secondaryZygoteState;
592        }
593
594        throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
595    }
596
```

#### 1.2 在zygote中是怎么创建的
在前面已经说了, zygot是由init进程创建的，启动起来后会调用ZygoteInit.main方法，创建socket管道  
然后runSelectLoop()，的带客户端发消息来创建进程

这些都在前面的章节中讲过，大致的伪代码
```java
public static void main(String argv[]) {
    try {
        runSelectLoop(abiList); //【见小节5】
        ....
    } catch (MethodAndArgsCaller caller) {
        caller.run(); //【见小节16】
    } catch (RuntimeException ex) {
        closeServerSocket();
        throw ex;
    }
}
```

查看runSelectLoop方法
```java
Runnable runSelectLoop(String abiList) {
174        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
175        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
176
177        fds.add(mServerSocket.getFileDescriptor());
178        peers.add(null);
179
180        while (true) {
181            StructPollfd[] pollFds = new StructPollfd[fds.size()];
182            for (int i = 0; i < pollFds.length; ++i) {
183                pollFds[i] = new StructPollfd();
184                pollFds[i].fd = fds.get(i);
185                pollFds[i].events = (short) POLLIN;
186            }
187            try {
188                Os.poll(pollFds, -1);
189            } catch (ErrnoException ex) {
190                throw new RuntimeException("poll failed", ex);
191            }
192            for (int i = pollFds.length - 1; i >= 0; --i) {
193                if ((pollFds[i].revents & POLLIN) == 0) {
194                    continue;
195                }
196
197                if (i == 0) {
                       //等待远程命令
198                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
199                    peers.add(newPeer);
200                    fds.add(newPeer.getFileDesciptor());
201                } else {
202                    try {
203                        ZygoteConnection connection = peers.get(i);
                           //获取到客户端发来的消息,来创建进程，看下面分析
204                        final Runnable command = connection.processOneCommand(this);
205
206                        if (mIsForkChild) {
207                            // We're in the child. We should always have a command to run at this
208                            // stage if processOneCommand hasn't called "exec".
209                            if (command == null) {
210                                throw new IllegalStateException("command == null");
211                            }
212
213                            return command;
214                        } else {
215                            // We're in the server - we should never have any commands to run.
216                            if (command != null) {
217                                throw new IllegalStateException("command != null");
218                            }
219
220                            // We don't know whether the remote side of the socket was closed or
221                            // not until we attempt to read from it from processOneCommand. This shows up as
222                            // a regular POLLIN event in our regular processing loop.
223                            if (connection.isClosedByPeer()) {
224                                connection.closeSocket();
225                                peers.remove(i);
226                                fds.remove(i);
227                            }
228                        }
229                    } catch (Exception e) {
230                        if (!mIsForkChild) {
231                            // We're in the server so any exception here is one that has taken place
232                            // pre-fork while processing commands or reading / writing from the
233                            // control socket. Make a loud noise about any such exceptions so that
234                            // we know exactly what failed and why.
235
236                            Slog.e(TAG, "Exception executing zygote command: ", e);
237
238                            // Make sure the socket is closed so that the other end knows immediately
239                            // that something has gone wrong and doesn't time out waiting for a
240                            // response.
241                            ZygoteConnection conn = peers.remove(i);
242                            conn.closeSocket();
243
244                            fds.remove(i);
245                        } else {
246                            // We're in the child so any exception caught here has happened post
247                            // fork and before we execute ActivityThread.main (or any other main()
248                            // method). Log the details of the exception and bring down the process.
249                            Log.e(TAG, "Caught post-fork exception in child process.", e);
250                            throw e;
251                        }
252                    } finally {
253                        // Reset the child flag, in the event that the child process is a child-
254                        // zygote. The flag will not be consulted this loop pass after the Runnable
255                        // is returned.
256                        mIsForkChild = false;
257                    }
258                }
259            }
260        }
261    }

```

调用ZygoteConnection 类里面的函数
```java
Runnable processOneCommand(ZygoteServer zygoteServer) {
124        String args[];
125        .....................................................
233
           //fork进程
234        pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
235                parsedArgs.runtimeFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
236                parsedArgs.niceName, fdsToClose, fdsToIgnore, parsedArgs.startChildZygote,
237                parsedArgs.instructionSet, parsedArgs.appDataDir);
238
239        try {
240            if (pid == 0) {
241                // in child
242                zygoteServer.setForkChild();
243
244                zygoteServer.closeServerSocket();
245                IoUtils.closeQuietly(serverPipeFd);
246                serverPipeFd = null;
247
248                return handleChildProc(parsedArgs, descriptors, childPipeFd,
249                        parsedArgs.startChildZygote);
250            } else {
251                // In the parent. A pid < 0 indicates a failure and will be handled in
252                // handleParentProc.
253                IoUtils.closeQuietly(childPipeFd);
254                childPipeFd = null;
255                handleParentProc(pid, descriptors, serverPipeFd);
256                return null;
257            }
258        } finally {
259            IoUtils.closeQuietly(childPipeFd);
260            IoUtils.closeQuietly(serverPipeFd);
261        }
262    }

```

在Zygote.java中

```java
public static int forkAndSpecialize(int uid, int gid, int[] gids, int runtimeFlags,
134          int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose,
135          int[] fdsToIgnore, boolean startChildZygote, String instructionSet, String appDataDir) {
136        VM_HOOKS.preFork();
137        // Resets nice priority for zygote process.
138        resetNicePriority();
139        int pid = nativeForkAndSpecialize(
140                  uid, gid, gids, runtimeFlags, rlimits, mountExternal, seInfo, niceName, fdsToClose,
141                  fdsToIgnore, startChildZygote, instructionSet, appDataDir);
142        // Enable tracing as soon as possible for the child process.
143        if (pid == 0) {
144            Trace.setTracingEnabled(true, runtimeFlags);
145
146            // Note that this event ends at the end of handleChildProc,
147            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "PostFork");
148        }
149        VM_HOOKS.postForkCommon();
150        return pid;
151    }
```

查看nativeForkAndSpecialize方法,调用到jni里面的方法com_android_internal_os_Zygote.cpp  

public static int nativeForkAndSpecialize() {
  ..............
  ForkAndSpecializeCommon();
  ..............
}
