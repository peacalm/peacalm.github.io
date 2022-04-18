---
title: "在线服务的异步RPC延时控制和无锁异步任务同步组件"
date: 2022-04-18T19:04:20+08:00
lastmod: 2022-04-18T19:04:20+08:00
draft: false
tags: ["多线程同步", "异步", "延时控制", "RPC"]
categories: ["技术"]
---

## 一、关于延时

### 控制延时是保证在线服务可用性的必要手段

在线服务对延时敏感。一般对在线服务发起远程调用时，都会配有一个超时限制，一旦请求超时，
则认为本次请求失败，服务不可用。因此控制延时是保证在线服务可用性的必要手段。

### RPC远程调用耗时的复杂性

网络环境是不可靠的，数据在网络中传输的耗时是不可控的，因此为了控制延时，
需要为每次socket发送和接受数据都配置一个超时限制。
而一次RPC远程调用可能需要执行多次socket请求，例如数据包很大、io状态不佳、需要重试等原因，
因此为单次socket请求设置的超时限制，并不能准确代表一次RPC远程调用的网络耗时。

因此，为了更准确控制超时，一些RPC框架内置了io线程池，采用异步方式进行socket网络请求，
超时控制会比同步模式准确一些。

不过，虽然异步请求是比同步请求模式更先进的控制超时的方式，但更复杂，
也引入了更多影响延时的因素和需要关注和调优的参数。
例如，异步模式虽然可以避免多次socket请求对超时控制的影响，但又引入了
io线程池的调度耗时、控制异步io超时的定时器的准确性等因素对总体RPC延时的影响。

再退一步，从业务客户端视角看，一次RPC远程调用的耗时，除了网络耗时，
还包括数据在本端（客户端）和远端（服务端）的序列化和反序列化耗时，当数据包过大时，
这一部分耗时和对CPU的消耗也是不能忽视的。

### 异步RPC是必要的

因此，使用异步RPC远程调用总是必要的，除了能使延时控制得更准确外，还可以并行执行多个RPC请求，
进一步降低延时，而并行执行一些相互独立的任务也是降低延时的非常重要的优化策略。

更进一步，除了RPC框架内置的异步网络请求模式外，还可以再退一层，站在业务的更大的视角上看，
在在线服务内建立线程池，用于执行异步任务，这些异步任务不仅可以包括RPC框架涵盖的网络请求，
还可以包含数据的序列化反序列化，以及业务上需要特殊处理的逻辑。
例如，假如一个业务上的异步任务其实内部需要先后访问两次远程服务，然后再做一些处理，
那么用一个业务上的异步任务给它一个整体上的超时，对业务上讲，控制延时将更准确。
或者，业务上的异步任务可以仅仅是对使用了同步RPC框架的一次RPC任务的封装，那么这个业务上设定的超时，
就包含了网络耗时、数据序列化反序列化耗时、业务特殊处理逻辑等所有步骤的耗时，
比单独使用RPC框架的超时，控制延时更准确。


## 二、关于同步

异步执行的程序之间总是需要同步的。

### 无锁同步

线程间同步一般常用锁，例如互斥锁、读写锁等，而"无锁"同步，一般就要直接使用原子标记，事实上，
锁的内部实现方式也是使用了原子标记。

### 异步任务同步

业务上给每一个异步任务都设定一个预期的超时时间，主线程发送出一个或多个异步任务后，
等待一段时间——这个时间应该是这些异步任务超时时间的最大值——让这些任务并行执行一会儿，
然后需要判断异步任务执行得怎么样了，是已经成功了？或是已经失败了？或是还在执行中？
如果还在执行中，那么主线程可以给这个任务标记为"已超时"，
让这个超时任务不要再影响主线程，或进入超时任务的特殊处理方法。

异步任务成功或失败的标记信息，是任务线程发送给主线程的信号，
而任务已超时的信息，是主线程发送给任务线程的信号。
这些就涉及到线程间同步问题。

### 异步任务无锁同步组件，C++为例

以下C++实现代码中 SyncKit类 就是一个简单的异步任务无锁同步组件，
它是主线程和异步任务线程共享的数据结构，分别为其定义了可调用成员方法。
主线程调用master_开头的方法，异步任务线程调用slave_开头的方法。

主线程使用方式很简单，发出异步任务后等待一段时间，然后调用master_check_ret()检查结果即可。
任务线程使用方式稍微复杂，主要要保证任务完成后将结果数据写入主线程提供的承载任务结果的数据结构时，
要保证主线程尚在等待，还没有标记超时。否则，如果主线程已经标记了超时，代表主线程可能已经开始使用
承载任务结果的数据结构了，如果此时任务线程还对这个数据结构进行修改，那么就会造成并发安全问题。
slave样例伪代码见下代码注释中。

```

// 用于保存异步任务执行结果的整型状态码。
// 正数为任务成功，负数为任务失败，0代表尚未完成，可被判成超时。
class TaskResultCode {
public:
  TaskResultCode() : ival_(0) {}

  int value() const { return ival_; }

  bool unready() const { return ival_ == 0; }
  bool succeeded() const { return ival_ > 0; }
  bool failed() const { return ival_ < 0; }

  template <int i = 1>
  void set_succeeded() {
    static_assert(i > 0, "success code should be positive");
    ival_ = i;
  }

  template <int i = -1>
  void set_failed() {
    static_assert(i < 0, "failure code should be negative");
    ival_ = i;
  }

private:
  std::atomic<int> ival_;
};


// 异步任务同步工具。主线程为master，任务线程为slave。
// 主线程发送异步任务时，打包一个SyncKit用作同步，同时还可以传入一个共享数据结构，
// 用于让slave把任务执行结果数据保存在此。
class SyncKit {
public:
  SyncKit() = default;

  // 主线程master检查异步任务执行结果，如果尚未完成则标记已经超时。
  const TaskResultCode& master_check_ret() {
    if (task_done_) {
      return ret_;
    }
    is_timeout_ = true;
    if (slave_checking_timeout_) {
      while (!task_done_) {}  // waiting
      return ret_;
    }
    return ret_;
  }

  // 主线程检master查任务结果，但不标记超时。
  const TaskResultCode& master_peek_ret() { return ret_; }

  /**
   * slave code example:
   *
   * if slave_test_timeout():
   *   return
   * if invalid parameters or conditions:
   *   slave_set_failed_unsafe<some failure code>
   *
   * slave starts to run a task。 e.g. do some RPC.
   * 
   * // 异步任务已经完成，准备将任务结果写回到与master共享的数据结构中。
   * // 写之前先判断master是否已经标记超时。
   * if slave_check_whether_timeout():
   *     return
   * 
   * // 尚未超时，master还在等待。此后的代码逻辑必须保证slave一定会调用
   * // slave_set_failed<failure code>()或slave_set_succeeded<success code>()
   * // 并最终对 task_done_ 赋值为true，否则可能阻塞主线程。
   * 
   * if task failed:
   *     slave writes failed result to shared data or does nothing.
   *     slave_set_failed<some failure code>()
   * else:
   *     slave writes succeeded result to shared data.
   *     slave_set_succeeded<some success code>()
   *
   */

  // 任务线程slave检查主线程master是否已经标记超时。如果已经标记为超时，
  // 则slave不能再对与master共享的数据结构做任何修改，因为此时master可能正在读取这些数据。
  // 任务线程slave在修改与master共享的数据结构之前，必须调用此方法确保尚未超时。
  bool slave_check_whether_timeout() {
    slave_checking_timeout_ = true;
    if (is_timeout_) {
      task_done_ = true;
      return true;
    } else {
      return false;
    }
  }

  bool slave_test_timeout() const { return is_timeout_; }

  template <int failure_code = -1>
  void slave_set_failed() {
    ret_.set_failed<failure_code>();
    task_done_ = true;
  }

  template <int success_code = 1>
  void slave_set_succeeded() {
    ret_.set_succeeded<success_code>();
    task_done_ = true;
  }

  // use default code 1 for succeeded and -1 for failed.
  void slave_set_ret(bool succeeded) {
    if (succeeded) {
      slave_set_succeeded();
    } else {
      slave_set_failed();
    }
  }

  // unsafe, could call this without calling slave_check_whether_timeout first.
  // 但是可能导致master观测到的TaskResultCode变化一次，从0变为负数。
  template <int failure_code = -1>
  void slave_set_failed_unsafe() {
    if (!is_timeout_) {
      ret_.set_failed<failure_code>();
    }
    task_done_ = true;
  }

private:
  std::atomic<bool> is_timeout_{false};
  std::atomic<bool> slave_checking_timeout_{false};
  std::atomic<bool> task_done_{false};
  TaskResultCode ret_;
};



// 以下是一些配套工具代码

typedef std::shared_ptr<SyncKit> SyncKitPtr;

inline SyncKitPtr new_sk() { return std::make_shared<SyncKit>(); }

struct SyncKitGuard {
  SyncKitPtr sk;
  std::string name;

  SyncKitGuard() : sk(new_sk()) {}

  SyncKitGuard(std::string n) : sk(new_sk()), name(std::move(n)) {}

  SyncKitGuard(std::string n, SyncKitPtr orther_sk) : sk(orther_sk), name(
      std::move(n)) { assert(sk != nullptr); }

  SyncKitGuard(SyncKitGuard&& r) : sk(std::move(r.sk)),
                                   name(std::move(r.name)) { r.sk = nullptr; }

  SyncKitGuard(const SyncKitGuard& r) = delete;

  ~SyncKitGuard() {
    if (sk) sk->master_check_ret();
  }
};

struct SyncKitGuardList : public std::list<SyncKitGuard> {
  SyncKitGuard& new_back(std::string name = "") {
    push_back(SyncKitGuard(std::move(name)));
    return back();
  }
};

struct SyncKitGuardMap : public std::map<std::string, SyncKitGuard> {};

```