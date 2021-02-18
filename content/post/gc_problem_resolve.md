---
title: "Java垃圾回收浅析(4)-GC常见问题分析"
date: 2019-02-27T11:00:00+08:00
categories: ["技术"]
tags: ["GC","Java"]
thumbnail: "images/gc_problem_resolve/yinxing.jpg"
draft: false
---
### 常见的几种GC问题
回顾一下：上面几篇先讲到了[java对象内存分配过程](https://coldwalker.github.io/2019/02//gc_object_alloc_process/)、然后总结了[几种GC方式和常见的GC算法原理](https://coldwalker.github.io/2019/02//gc_intro/)，也基本了解了[GC日志怎么看](https://coldwalker.github.io/2019/02//gc_log_analyze/)，接下来该是解决问题的时候了，本文会结合GC日志分析几种比较常见的GC问题，从问题引发原因到具体解决方案，希望能从中有所收获！

#### 定位GC日志中STW时间较长的行为
打印所有STW停顿：
```
-XX:+PrintGCApplicationStoppedTime和-XX:+PrintGCApplicationConcurrentTime
```
<i>找出所有暂停时间超过阈值的地方，如下：</i>
```
awk '{match($0,/.*stopped:(.*)seconds,/,a);if(a[1]>0.5) print $0}' /data1/proxy_web_release/gclogs/gc.log.20170617_015811
```

#### promotion failed
GC日志中有时候会发现如下类似的带有“promotion failed”的日志，伴随着这类日志一般都会有较长的STW暂停时间，因此也会对线上应用造成较大影响。接下来对“promotion failed”这种GC异常情况进行一下分析：
<i>一个常见的promotion failed的gc日志示例：</i>
```
2017-06-17T23:33:20.381+0800: 77708.486: [GC (Allocation Failure) 2017-06-17T23:33:20.382+0800: 77708.487: [ParNew (promotion failed): 3774912K->3679034K(3774912K), 0.2991519 secs]2017-06-17T23:33:20.681+0800: 77708.786: [CMS: 3270027K->3522738K(6291456K), 1.9341892 secs] 6332035K->3522738K(10066368K), [Metaspace: 71553K->71553K(1116160K)], 2.2340311 secs] [Times: user=5.61 sys=0.06, real=2.24 secs]
2017-06-17T23:33:22.616+0800: 77710.721: Total time for which application threads were stopped: 2.2411722 seconds, Stopping threads took: 0.0002553 seconds
```
<i>young gc 在2017-06-17T23:33:20.382触发，young gc过程花费时间为0.2991519 s，回收前后整个young区占用内存从3774912K变为3679034K。随后触发了一次Full GC来对整个堆进行STW的回收，年老代GC回收花了1.9341892 s，回收后年老代空间从3270027K变成3522738K（整个年轻代对象都promote到年老代去了），整个JVM堆空间从6332035K降低到3522738K。整个Full GC花费的时间是 2.24s（业务线程暂停时间）。</i>

##### promotion failed发生的场景
young GC时，对象需要从年轻代提升到年老代，但年老代可用空间由于各种原因存放不下这些对象，这时会抛出promotion failed，然后触发一次Full GC来对年老代和永久代（metaspace）进行回收，所以发生promotion failed时是会暂停业务线程引起停顿的，需要特别留意。

##### 什么情况下对象会从年轻代往年老代提升呢？
_主要有以下几种：_

* 年轻代中对象到达一定年龄的对象在minor gc时会向年老代提升。
* minor gc时一个survivor空间无法装下所有年轻代存活的对象时，部分未到达年龄的对象也会向年老代提前提升。

##### Young GC的“悲观策略”
有的情况下，如果JVM判断本次young GC需要提升的大小年老代放不下就会放弃本次young GC，而是直接触发一次Full GC。这种情况被称为young GC的”悲观策略“。ParNew收集器里”悲观策略“相关判断逻辑在ParNewGeneration的collect里，collection_attempt_is_safe定义在基类DefNewGeneration里，将真正的判断逻辑_next_gen->promotion_attempt_is_safe交给了next_gen（也就是CMS回收器），promotion_attempt_is_safe代码在ConcurrentMarkSweepGeneration里。

```
// If the next generation is too full to accommodate worst-case promotion
  // from this generation, pass on collection; let the next generation
  // do it.
  if (!collection_attempt_is_safe()) {
    gch->set_incremental_collection_failed();  // slight lie, in that we did not even attempt one
    return;
  }
```

```
bool DefNewGeneration::collection_attempt_is_safe() {
  if (!to()->is_empty()) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print(" :: to is not empty :: ");
    }
    return false;
  }
  if (_next_gen == NULL) {
    GenCollectedHeap* gch = GenCollectedHeap::heap();
    _next_gen = gch->next_gen(this);
  }
  return _next_gen->promotion_attempt_is_safe(used());
}

```
###### CMS预测本次young gc需要promote的大小老年代是否可以放下的条件是：
1. 老年代可用空间大于gc_stats统计的新生代每次平均晋升的大小。
2. 老年代可以容纳目前新生代的所有对象。

*两个条件满足一个即可正常触发young GC。*
```
bool ConcurrentMarkSweepGeneration::promotion_attempt_is_safe(size_t max_promotion_in_bytes) const {
  size_t available = max_available();
  size_t av_promo  = (size_t)gc_stats()->avg_promoted()->padded_average();
  bool   res = (available >= av_promo) || (available >= max_promotion_in_bytes);
  if (Verbose && PrintGCDetails) {
    gclog_or_tty->print_cr(
      "CMS: promo attempt is%s safe: available(" SIZE_FORMAT ") %s av_promo(" SIZE_FORMAT "),"
      "max_promo(" SIZE_FORMAT ")",
      res? "":" not", available, res? ">=":"<",
      av_promo, max_promotion_in_bytes);
  }
  return res;
}
```

如下这次GC情况就是触发了young GC的“悲观策略”，实际这次young GC没有执行，而是直接进行了一次Full GC。：
```
2018-09-24T20:18:01.761+0800: 274170.361: [GC (Allocation Failure) 2018-09-24T20:18:01.762+0800: 274170.362: [ParNew: 6291456K->6291456K(7864320K), 0.0000411 secs] 18389797K->18389797K(20447232K), 0.0011756 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2018-09-24T20:18:01.763+0800: 274170.362: [Full GC (Allocation Failure) 2018-09-24T20:18:01.763+0800: 274170.363: [CMS: 12098341K->12111250K(12582912K), 6.4734175 secs] 18389797K->12111250K(20447232K), [Metaspace: 82207K->82207K(1124352K)], 6.4746029 secs] [Times: user=4.90 sys=1.03, real=6.48 secs]
2018-09-24T20:18:08.238+0800: 274176.838: Total time for which application threads were stopped: 6.5035487 seconds, Stopping threads took: 0.0008020 seconds
```
_因此，当GC log出现promotion failed时肯定是不满足”悲观策略“的判断进行了并行的新生代的垃圾回收，在最后往年老代提升空间不够时才报出的。_


##### promotion failed时触发的Full GC的两种情况
采用CMS作为老年代垃圾回收器的时候，当发生promotion failed时会触发一次Full GC，但这次Full GC可能存在两种情况。JVM根据某些条件判断本次Full GC是否需要“整理”来决定这次Full GC是使用单线程的带标记整理（mark-sweep-compact）的Serial GC的算法（do_compaction_work）来进行整个堆的垃圾回收，还是使用CMS自己的mark-sweep（do_mark_sweep_work）来做一次 多线程的*foregroud GC* 来对老年代进行回收。

_是否需要整理的判断条件如下:_
1. UseCMSCompactAtFullCollection参数开启（默认开启）。
2. 上次正常的backgroud GC后Full GC（全局Full GC或者老年代foreground GC）的次数_full_gcs_since_conc_gc（每次background GC执行完sweeping阶段就会设置为0）达到启动参数设置的阈值CMSFullGCsBeforeCompaction（默认是0，也就是默认就是每次都“compact”）。
3. 如果是用户触发的System GC，那么直接进行compact。
4. 如果上一次young gc晋升时失败了(incremental_collection_failed为true)或者“预测本次晋升可能失败”，那么也直接进行compact。

所以实际上发现，能走到并行的foregroud GC的条件比较苛刻，默认情况下都是执行单线程带compact的Serial Old GC算法（Lisp2算法实现，在genMarkSweep.cpp中）来对整个堆进行回收。

<i>采用并行foreground gc还是serial old gc的判断</i>
```
*should_compact =
    UseCMSCompactAtFullCollection &&
    ((_full_gcs_since_conc_gc >= CMSFullGCsBeforeCompaction) ||
     GCCause::is_user_requested_gc(gch->gc_cause()) ||
     gch->incremental_collection_will_fail(true /* consult_young */));
  ...
 bool incremental_collection_will_fail(bool consult_young) {
  
    assert(heap()->collector_policy()->is_two_generation_policy(),
           "the following definition may not be suitable for an n(>2)-generation system");
    return incremental_collection_failed() ||
           (consult_young && !get_gen(0)->collection_attempt_is_safe());
  }
```
######  MSC实现的Serial Old GC
<i>如果需要压缩就使用“带压缩”算法的单线程的Serial Old GC来进行整堆的Full GC，否则使用CMS自己的多线程的foreground GC来对老年代进行回收。</i>
```
if (should_compact) {
    // If the collection is being acquired from the background
    // collector, there may be references on the discovered
    // references lists that have NULL referents (being those
    // that were concurrently cleared by a mutator) or
    // that are no longer active (having been enqueued concurrently
    // by the mutator).
    // Scrub the list of those references because Mark-Sweep-Compact
    // code assumes referents are not NULL and that all discovered
    // Reference objects are active.
    ref_processor()->clean_up_discovered_references();

    if (first_state > Idling) {
      save_heap_summary();
    }

    do_compaction_work(clear_all_soft_refs);

    // Has the GC time limit been exceeded?
    DefNewGeneration* young_gen = _young_gen->as_DefNewGeneration();
    size_t max_eden_size = young_gen->max_capacity() -
                           young_gen->to()->capacity() -
                           young_gen->from()->capacity();
    GenCollectedHeap* gch = GenCollectedHeap::heap();
    GCCause::Cause gc_cause = gch->gc_cause();
    size_policy()->check_gc_overhead_limit(_young_gen->used(),
                                           young_gen->eden()->used(),
                                           _cmsGen->max_capacity(),
                                           max_eden_size,
                                           full,
                                           gc_cause,
                                           gch->collector_policy());
  } else {
    do_mark_sweep_work(clear_all_soft_refs, first_state,
      should_start_over);
  }
```

<i>MSC的具体实现在GenMarkSweep类中（genMarkSweep.cpp) </i>
```
//concurrentMarkSweepGeneration.cpp
void CMSCollector::do_compaction_work(bool clear_all_soft_refs) {
...
GenMarkSweep::invoke_at_safepoint(_cmsGen->level(),
    ref_processor(), clear_all_soft_refs);
...
//genMarkSweep.cpp
void GenMarkSweep::invoke_at_safepoint(int level, ReferenceProcessor* rp, bool clear_all_softrefs) {
  guarantee(level == 1, "We always collect both old and young.");
  assert(SafepointSynchronize::is_at_safepoint(), "must be at a safepoint");

  GenCollectedHeap* gch = GenCollectedHeap::heap(); //处理整个堆
  ...
```
######  并行的foreground GC
<i>如果使用并行的foreground GC，整个过程都是暂停应用的，而且是*同步*的（但仍然是多线程进行处理），为了提高效率，会跳过其中一些阶段。那么这些省下来的阶段主要是并行阶段：Precleaning、AbortablePreclean，Resizing。
另外，如果当前backgroud的GC正在进行中，如果走到了foreground GC，那么foreground GC会在下一个安全点接管未完成的backgroud GC的后续步骤，这样还能跳过一些之前backgroud GC已经完成的阶段。</i>

concurrentMarkSweepGeneration.cpp中相关代码如下：
```
// A work method used by the foreground collector to do
// a mark-sweep, after taking over from a possibly on-going
// concurrent mark-sweep collection.
void CMSCollector::do_mark_sweep_work(bool clear_all_soft_refs,
  CollectorState first_state, bool should_start_over) {
  if (PrintGC && Verbose) {
    gclog_or_tty->print_cr("Pass concurrent collection to foreground "
      "collector with count %d",
      _full_gcs_since_conc_gc);
  }
  switch (_collectorState) {
    case Idling:
      if (first_state == Idling || should_start_over) {
        // The background GC was not active, or should
        // restarted from scratch;  start the cycle.
        _collectorState = InitialMarking;
      }
      // If first_state was not Idling, then a background GC
      // was in progress and has now finished.  No need to do it
      // again.  Leave the state as Idling.
      break;
    case Precleaning:
      // In the foreground case don't do the precleaning since
      // it is not done concurrently and there is extra work
      // required.
      _collectorState = FinalMarking;
  }
  collect_in_foreground(clear_all_soft_refs, GenCollectedHeap::heap()->gc_cause());

  // For a mark-sweep, compute_new_size() will be called
  // in the heap's do_collection() method.
}
```
```
void CMSCollector::collect_in_foreground(bool clear_all_soft_refs, GCCause::Cause cause) {
switch (_collectorState) {
      case InitialMarking:
        register_foreground_gc_start(cause);
        init_mark_was_synchronous = true;  // fact to be exploited in re-mark
        checkpointRootsInitial(false);
        assert(_collectorState == Marking, "Collector state should have changed"
          " within checkpointRootsInitial()");
        break;
      case Marking:
        // initial marking in checkpointRootsInitialWork has been completed
        if (VerifyDuringGC &&
            GenCollectedHeap::heap()->total_collections() >= VerifyGCStartAt) {
          Universe::verify("Verify before initial mark: ");
        }
        {
          bool res = markFromRoots(false);
          assert(res && _collectorState == FinalMarking, "Collector state should "
            "have changed");
          break;
        }
      case FinalMarking:
        if (VerifyDuringGC &&
            GenCollectedHeap::heap()->total_collections() >= VerifyGCStartAt) {
          Universe::verify("Verify before re-mark: ");
        }
        checkpointRootsFinal(false, clear_all_soft_refs,
                             init_mark_was_synchronous);
        assert(_collectorState == Sweeping, "Collector state should not "
          "have changed within checkpointRootsFinal()");
        break;
      case Sweeping:
        // final marking in checkpointRootsFinal has been completed
        if (VerifyDuringGC &&
            GenCollectedHeap::heap()->total_collections() >= VerifyGCStartAt) {
          Universe::verify("Verify before sweep: ");
        }
        sweep(false);
        assert(_collectorState == Resizing, "Incorrect state");
        break;
      case Resizing: {
        // Sweeping has been completed; the actual resize in this case
        // is done separately; nothing to be done in this state.
        _collectorState = Resetting;
        break;
      }
      case Resetting:
        // The heap has been resized.
        if (VerifyDuringGC &&
            GenCollectedHeap::heap()->total_collections() >= VerifyGCStartAt) {
          Universe::verify("Verify before reset: ");
        }
        save_heap_summary();
        reset(false);
        assert(_collectorState == Idling, "Collector state should "
          "have changed");
        break;
      case Precleaning:
      case AbortablePreclean:
        // Elide the preclean phase
        _collectorState = FinalMarking;
        break;
      default:
        ShouldNotReachHere();
    }
}
```
_foreground GC由于没有压缩导致本次GC完还是没有足够空间存放的补救_
很多情况下，如果走到了foreground GC没有进行compact，碎片问题还是没法得到解决。因此如果这次foreground GC后还是空间不足的话，就会接着进行一次彻底的Full GC，并清理软引用来尽可能回收内存，标记回收软引用后，CMS就会将is_compact设置成true，这样这最后一次Full GC就会使用单线程（VM Thread）的带标记整理（mark-sweep-compact）的Serial Old GC的算法（do_compaction_work）来进行整个堆的垃圾回收。如果这次Full GC还是不行就抛出OOM了。
```
// Try a full collection; see delta for bug id 6266275
    // for the original code and why this has been simplified
    // with from-space allocation criteria modified and
    // such allocation moved out of the safepoint path.
    gch->do_collection(true             /* full */,
                       false            /* clear_all_soft_refs */,
                       size             /* size */,
                       is_tlab          /* is_tlab */,
                       number_of_generations() - 1 /* max_level */);
  }

  result = gch->attempt_allocation(size, is_tlab, false /*first_only*/);

  if (result != NULL) {
    assert(gch->is_in_reserved(result), "result not in heap");
    return result;
  }

  // OK, collection failed, try expansion.
  result = expand_heap_and_allocate(size, is_tlab);
  if (result != NULL) {
    return result;
  }

  // If we reach this point, we're really out of memory. Try every trick
  // we can to reclaim memory. Force collection of soft references. Force
  // a complete compaction of the heap. Any additional methods for finding
  // free memory should be here, especially if they are expensive. If this
  // attempt fails, an OOM exception will be thrown.
  {
    UIntFlagSetting flag_change(MarkSweepAlwaysCompactCount, 1); // Make sure the heap is fully compacted

    gch->do_collection(true             /* full */,
                       true             /* clear_all_soft_refs */,
                       size             /* size */,
                       is_tlab          /* is_tlab */,
                       number_of_generations() - 1 /* max_level */);
  }
```
```
if (clear_all_soft_refs && !*should_compact) {
    // We are about to do a last ditch collection attempt
    // so it would normally make sense to do a compaction
    // to reclaim as much space as possible.
    if (CMSCompactWhenClearAllSoftRefs) {
      // Default: The rationale is that in this case either
      // we are past the final marking phase, in which case
      // we'd have to start over, or so little has been done
      // that there's little point in saving that work. Compaction
      // appears to be the sensible choice in either case.
      *should_compact = true;
    }
```

##### promote failed后GC日志打印的占用大小释疑
当出现promote failed时，有的情况下会发现，本次触发的young GC后，年轻代的内存占用比回收前还上涨了。如下面示例这次：
这次Minor GC后，居然年轻代的占用空间还从3088943K涨到了3476337K。这个需要了解下Minor GC的过程：Minor GC开始后，遍历GC Roots，碰到eden区和from区的对象就把它挪到to区，但是并不会马上释放老的对象，直到GC完成后才会释放。如果GC出现晋升失败，这种情况下是不会释放这些老对象的，尽管to区里已经有一些eden区和from区对象的副本了。但是GC不管是否成功都会在结束后对from区和to区进行切换，这个时候原来的to区变成from区了，由于GC日志统计的只是eden+from的使用大小，GC前统计的eden+from变成了eden（未释放老对象）+原来的to（包含eden存活的部分拷贝+原来from存活的备份拷贝），这样是有可能统计出来比之前要大的。

```
2017-10-13T18:11:03.817+0800: 341300.509: [GC (Allocation Failure) 2017-10-13T18:11:03.818+0800: 341300.509: [ParNew (promotion failed): 3088943K->3476337K(3495296K), 0.2446697 secs]2017-10-13T18:11:04.063+0800: 341300.754: [CMS: 3953608K->2518422K(6291456K), 1.4883145 secs] 7042552K->2518422K(9786752K), [Metaspace: 73035K->73035K(1116160K)], 1.7337265 secs] [Times: user=4.32 sys=0.02, real=1.73 secs]
```

#### concurrent mode failure
有的时候GC日志中还会出现“concurrent mode failure”类似的异常，一般这种异常会伴随着较长的STW时间发生。出现这种情况的前提是使用了CMS作为年老代的垃圾回收器。CMS是一款并发收集器，GC回收线程和业务的用户线程是并发执行的（除了初始标记和重新标记外其他阶段都是可以和业务线程并行的），并发执行意味着在垃圾回收执行的同时还会不停有新的对象promote到年老代。在并发周期执行期间，用户的线程依然在运行，如果这时候如果应用线程向老年代请求分配的空间超过剩余的空间（担保失败），就会触发concurrent mode failure。

```
2018-05-05T15:32:56.818+0800: 101200.681: [GC (Allocation Failure) 2018-05-05T15:32:56.819+0800: 101200.682: [ParNew: 5242832K->5242832K(5242880K), 0.0000388 secs]2018-05
-05T15:32:56.819+0800: 101200.682: [CMS2018-05-05T15:32:56.873+0800: 101200.736: [CMS-concurrent-sweep: 0.212/0.240 secs] [Times: user=0.26 sys=0.00, real=0.23 secs]
 (concurrent mode failure): 7033555K->7033541K(9437184K), 0.0766034 secs] 12276388K->12276374K(14680064K), [Metaspace: 75013K->75013K(1118208K)], 0.0777029 secs] [Times: user=0.00 sys=0.00, real=0.08 secs]
```
 
如上面这次concurrent mode failure，对象分配失败请求一次年轻代GC，但是这次年轻代GC并没有真正执行，根据历史promote的大小和当前年老代的空间剩余大小估算，剩余年老代空间可能不够存放promote上来的对象。因此直接触发了一次Full GC，由于当时backgroup GC正在进行中，所以这时会中止backgroup gc，然后执行一次Full GC或者foreground GC。如这次Full GC总共花费了0.08s，最终年老代空间从7033555K调整成7033541K，整个过程是STW的。

##### 导致concurrent mode failure的几种情况
当JVM需要申请老年代空间但剩余可用空间不够时，会去判断当前年老代是否正在进行中，如果是在进行中，除了执行Full GC外还会报告concurrent mode failure，以下几种情况会导致concurrent mode failure：
1. 年轻代发生young GC需要promote对象到老年代但老年代可用空间不够，且老年代正在执行background GC。
2. 年轻代由于“悲观策略”放弃young GC需要触发Full GC或者CMS的foreground GC，且老年代正在执行background GC。
3. 新分配对象大小超过-XX:PretenureSizeThreshold阈值直接在老年代分配但老年代可用空间不够，且老年代正在执行background GC。

<i>所以promotion failed不一定会导致concurrent mode failure；当然，concurrent mode failure也不一定都是由于promotion failed后的Full GC导致的。</i>
```
CMSCollector::acquire_control_and_collect
...
  if (first_state > Idling) {
    report_concurrent_mode_interruption();
  }
```

<i>如下日志：本次“concurrent mode failure”就是由于young GC时“promotion failed”之后的Full GC导致的，而对象直接在年老代分配但老年代可用空间不够时，：</i>

```
2017-07-13T21:37:21.616+0800: 2317149.720: [GC (Allocation Failure) 2017-07-13T21:37:21.616+0800: 2317149.721: [ParNew (promotion failed): 3774912K->3702108K(3774912K), 0.3012695 secs]2017-07-13T21:37:21.918+0800: 2317150.022: [CMS2017-07-13T21:37:21.929+0800: 2317150.034: [CMS-concurrent-abortable-preclean: 3.608/4.392 secs] [Times: user=0.00 sys=0.00, real=4.39 secs]
 (concurrent mode failure): 5408872K->5935072K(6291456K), 2.6779990 secs] 8578988K->5935072K(10066368K), [Metaspace: 72542K->72542K(1116160K)], 2.9802519 secs] [Times: user=0.00 sys=0.00, real=2.98 secs]
```

<i>如下日志：对象在年轻代分配但年轻代由于“悲观策略”并不真正执行而是直接触发一次Full GC。</i>

```
2018-05-05T15:32:56.818+0800: 101200.681: [GC (Allocation Failure) 2018-05-05T15:32:56.819+0800: 101200.682: [ParNew: 5242832K->5242832K(5242880K), 0.0000388 secs]2018-05
-05T15:32:56.819+0800: 101200.682: [CMS2018-05-05T15:32:56.873+0800: 101200.736: [CMS-concurrent-sweep: 0.212/0.240 secs] [Times: user=0.26 sys=0.00, real=0.23 secs]
 (concurrent mode failure): 7033555K->7033541K(9437184K), 0.0766034 secs] 12276388K->12276374K(14680064K), [Metaspace: 75013K->75013K(1118208K)], 0.0777029 secs] [Times: user=0.00 sys=0.00, real=0.08 secs]
```
<i>promotion failed也不一定会导致concurrent mode failure。如果当前CMS的background gc没有在执行的话就不会出现“concurrent mode failure”，而是显示Full GC。</i> 
 ```
2018-09-24T20:18:01.761+0800: 274170.361: [GC (Allocation Failure) 2018-09-24T20:18:01.762+0800: 274170.362: [ParNew: 6291456K->6291456K(7864320K), 0.0000411 secs] 18389797K->18389797K(20447232K), 0.0011756 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2018-09-24T20:18:01.763+0800: 274170.362: [Full GC (Allocation Failure) 2018-09-24T20:18:01.763+0800: 274170.363: [CMS: 12098341K->12111250K(12582912K), 6.4734175 secs] 18389797K->12111250K(20447232K), [Metaspace: 82207K->82207K(1124352K)], 6.4746029 secs] [Times: user=4.90 sys=1.03, real=6.48 secs]
```

#### 导致promotion failed和concurrent mode failure的原因和解决方案
##### 原因1：CMS触发太晚
###### 说明
CMS的backgroup GC触发时机太晚，会导致在backgroup GC完成前，年老代剩余可用空间放不下新提升上来的或者直接在年老代分配的对象，从而触发Full GC并中断并发过程。
###### 解决方案
通过参数-XX:CMSInitiatingOccupancyFraction=N -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSWaitDuration=M 调小，更早、更快触发年老代backgroup GC。
CMSInitiatingOccupancyFraction默认值是92%，CMS backgroup GC扫描间隔默认是：2000（2s）。
*但注意这里并不是调的越小越好，越小年老代GC频率会越高，整体业务暂停的时间有可能会更长。另外，某些情况下，该参数设置太小导致年老代空间触发backgroup GC前可用的空间根本放不下所有晋升上来的长生命周期的对象，从而导致JVM一直不停地做年老代GC，严重影响业务性能。*

##### 原因2：CMS GC回收处理效率太低
###### 说明
CMS是一个并发的垃圾回收器，用于CMS各个阶段的GC线程数默认值是： ConcGCThreads = （ParallelGCThreads+3）/4。
而这里ParallelGCThreads表示的是GC并行时使用的线程数。比如如果新生代使用ParNew，那么ParallelGCThreads也就是新生代GC线程数。默认情况下，当CPU数量小于8时，ParallelGCThreads的值就是CPU的数量，当CPU数量大于8时，ParallelGCThreads的值等于3+5*cpuCount/8。
例如，在32核机器上，新生代并行GC线程数为 3 + 5*32/8 = 23，所以对应的CMS的并发线程数为 （23 +3） / 4 = 6。
###### 解决方案
通过参数 -XX:ParallelGCThreads和-XX：ConcGCThreads来分别增加年轻代GC并行处理和年老代GC并发处理的能力。但这里也并不是越大越好，因为CMS的很多阶段都是和业务线程并发进行的，如果用于GC的线程数太多也会更多抢占业务线程的处理时间片，从而影响业务性能，所以这里需要进行实际场景的验证测试。

##### 原因3:年老代空间碎片太多
###### 说明
CMS收集器采用的标记-清除算法，并不对年老代进行回收后的内存整理（虽然GC后会进行一些连续空间的合并）。因此多次GC后会存在较多的空间碎片。所以可能导致年老代剩余空间足够，但在大对象提升时由于没有连续的可用空间导致提升失败。
###### 解决方案
通过-XX:+UseCMSCompactAtFullCollection和-XX:CMSFullGCsBeforeCompaction=n来让JVM在多少次Full GC后进行年老代空间的碎片整理。但是这两个参数起到的是”病后用药“的作用，前提是已经发生了Full GC才触发（所以有一些比较曲线救国的办法就是在凌晨低峰期间：代码中定时调用System.gc来触发一次Full GC从而进行年老代的空间整理）。如果碎片问题确实比较严重，可以考虑改用G1垃圾回收器（G1每次GC都会对region进行整理）。

##### 原因4：年轻代提升速度过快
###### 说明
不管是promotion failed还是concurrent mode failure，都是由于提升时年老代可用空间不够导致的，因此提升速度过快是导致这些问题的直接原因。具体会导致提升过快的非业务原因有几点：
1. 年轻代对象晋升年龄阈值太小。
2. eden区太小，触发年轻代GC太容易。
3. survivor空间溢出（未满年龄的对象提前晋升，参考前面的Desired survivor size的计算）。
4. 业务中大对象较多，超过阈值直接在年老代中分配。
###### 解决方案
1. 调整-XX:MaxTenuringThreshold（默认15），提高年轻代晋升年龄。
注意：这里并不代表真正达到这个年龄才晋升，但JVM计算一个desired survivor size大小，survivor区对象如果累计到某一个age值的对象大小大于desired survivor size，下次晋升时大于等于该年龄的对象就会被提前promote到年老代，这里这个age值就是动态计算出来的。
2. eden区和survior可用区默认比例是（8:1)，survior区分s0和s1两部分，同一时刻只有一个区可用。所以实际可用大小为xmn的1/10。一般可以通过扩大eden区来减少年轻代GC次数，让年轻代对象到达触发年龄的速度慢一点。调整eden去和survior比例使用： -XX:SurvivorRatio=N。
3. survivor空间溢出的直接原因是计算的Desired survivor size太小，导致很多未满年龄的对象提前晋升到年老代，可以通过降低-XX:TargetSurvivorRatio=N（默认50，就是一个s0或者s1的一半），尽量避免提前晋升的溢出问题。
4. 针对大对象，可以通过-XX:PretenureSizeThreshold来设置直接在年老代分配的对象阈值，默认是0，即：由JVM动态决定（PS：这个参数尽量不要用，除非对自己的业务细节访问模型很清楚）。

*最后：如果以上调整都没办法彻底解决GC问题，那么应该考虑在系统内存可用范围内扩大整个heap的大小了。*

* * *

#### 延伸阅读：如何手动触发Full GC？
```
没有开启-XX:+DisableExplicitGC的前提下调用System.gc()就会发生FullGC
System.gc();
```
```
或者通过jmap命令触发：
jmap -histo:live pid
```

* * *
参考：
[HandlePromotionFailure](https://hllvm-group.iteye.com/group/topic/42365)
[GC算法之三 标记-压缩算法](https://zhuanlan.zhihu.com/p/51469246)