CPP Currency in Action note(4) Synchronizing concurrnet operation
===================================================================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

*2014-12-17*

这个章节包括内容：

* 等待一个事件
* 等待未来的一次性事件
* 加时间约束的等待
* 使用同步操作来简化代码

在上一章，我们看到了很多保护共享数据的方法。 但是，有时你不光需要保护数据，而且要协同不同线程中的操作。
比如，一个线程需要等待其他线程先完成一个任务之后再运行，或者需要等待一个事件或者条件。
C++标准库中提供了线程协同的工具，比如 `condition variables` 和 `futures` 。

在这个章节，我们将会讨论如何利用 `condition variables` 和 `futures` 来等待事件，或者如何使用他们来简化协同操作。

Waiting for an event or other condition
---------------------------------------------
想象你夜里坐一辆火车，不想坐过站。
一种方式是整夜醒着，并且留意火车到哪儿了，但是你会很累。
当然，你也可以查看下列车时刻表，查下列车到站时间，定下闹钟。
这样，你就不会错过站了，但是如果火车晚点了，那你就会在到站前提前醒了。
又或者，你的闹钟没电了，然后你会睡过头。 
最理想的方式就是，你睡觉就是，快到站的时候列车员来叫你（国内卧铺已经提供这种先进的服务了）。

那么，类比到多线程持续，如果要让一个线程在等待另外一个线程完成某个任务，同样有几种选项。
第一种方法是，让执行任务的那个线程设定一个标识，当它完成那个任务时，就修改标记，而另外一个线程不断查看那个标记。
这种方法是非常耗费资源的，持续检测标记会耗费CPU时间，此外，当mutex被等待线程中的一个锁定时，其他等候的线程都不能锁定。
这种方法就像前面的，整夜醒着等待火车到站。

第二种方法是让等待的线程每次检测完标记，用 `std::this_thread::sleep_for()` 睡眠一段时间。

.. code-block:: c++
    :linenos:

    void wait_for_flag()
    {
        std::unique_lock<std::mutex> lk(m);
        while(!flag)
        {
            lk.unlock();
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            lk.lock();
        }
    }

这是个巨大的进步，因为等候线程在sleep时，不会耗费CPU计算资源。
但是，设定一个合理的sleep时间是很难的；
太短的话，等候线程会检测标记的频率会比较高，耗费CPU资源；
太长的话，任务完成时，等候线程可能还睡着，导致延迟。

第三种同时也是很推荐的方法是，使用C++标准库中提供的机制来等待事件。
用来等候另外一个线程促发一个事件的最基础的机制是 `condition variable` 。
一个条件变量可以被关联到一些事件或者其他条件，一个或多个线程都在等待那个条件被满足。
当一些线程觉得条件满足了，就可以 `notify` 一个或多个等待此条件的线程，来唤醒它们并且允许其继续运行。

Waiting for a condition with condition variables
---------------------------------------------------
C++标准库提供了条件变量的两种实现： `std::condition_variable` 和 `std::condition_variable_any` 。
它们都被声明在 `<condition_variable>` 头文件中。
两者都需要一个mutex来实现功能，
不同的是，前者必须使用 `std::mutex` ，
然而，后者可以使用任何支持最基本mutex机制的类型，所以有一个 `_any` 后缀。
因为 `std::condition_variable_any` 比 `std::condition_variable` 有更大的灵活性，所以额外的消耗也是难以避免的。 
所以，推荐使用 `std::condition_variable` ，除非必须灵活性，才考虑使用 `std::condition_variable_any` 。

下面是一个具体应用的例子：

**Listing 4.1 Waiting for data to process with a std::condition_variable**

.. code-block:: c++
    :linenos:

    std::mutex mut;
    std::queue<data_chunk> data_queue;
    std::condition_variable data_cond;

    void data_preparation_thread()
    {
        while(more_data_to_prepare())
        {
            data_chunk const data = prepare_data();
            std::lock_guard<std::mutex> lk(mut);
            data_queue.push(data);
            data_cond.notify_one();
        }
    }

    void data_processing_thread()
    {
        while(true)
        {
            std::unique_lock<std::mutex> lk(mut);
            data_cond.wait(
                    lk, []{return !data_queue.empty();});
            data_chunk data = data_queue.front();
            data_queue.pop();
            lk.unlock();
            process(data);
            if(is_last_chunk(data))
                break;
        }
    }

首先，你用一个队列来在两个线程间传递数据。
当数据准备完毕，准备数据的线程会加锁，然后将数据压入队列中。
之后，它会调用 `notify_one()` 来唤醒一个 `std::condition_variable` 等候列表里的线程（如果存在的话）。

在另外一方面，你有个处理线程。
这个线程首先加锁，这个时候使用的是 `std::unique_lock` 而不是 `std::lock_guard` 。
这个线程之后在条件变量 `data_cond` 上调用了 `wait()` ，并传入了一个lambda函数 `[]{ return !data_queue.empty()` 来检查是否唤醒的条件满足。
如果条件不满足，那么 `wait()` 会解锁，然后将这个线程阻塞在等候队列中。
当条件变量被其他线程调用了 `notify_one()` ，在休眠的等候线程就会被唤醒，同时检查唤醒条件(前面的lambda函数)，如果条件满足，那么加锁运行；否则解锁，继续等待。
这就是为什么中间使用了 `std::unique_lock` ，因为线程可能还需要自己 `unlock` 锁。 

在调用 `wait()` 的过程中，一个条件变量可以无限次检查条件。
在检查条件时，它会加锁，wait()只会在条件满足时立刻返回，否则会一直阻塞线程向下执行。

使用队列来实现线程间的数据传递是一个非常常用的场景。 
如果做的足够好，同步操作只会限制在队列本身，这个会减少同步操作的数目以及race conditions的可能。
下面我们会实现一个通用的线程安全队列。

Building a thread-safe queue with condition variables
--------------------------------------------------------
当你尝试去设计一个通用的队列时，首先有必要花几分钟思考下需要准备那些操作。
下面让我们参考下C++标准库中的设计。

**Listing 4.2 std::queue interface**

.. code-block:: c++
    :linenos:

    template <class T, class Container = std::deque<T> >
    class queue {
        public:
            explicit queue(const Container&);
            ...

            void swap(queue  &q);
            bool empty() const;
            size_type size() const;

            T& front();
            const T& front() const;
            T& back();
            const T& back() const;

            void push(const T& x);
            void push(T&& x);
            void pop();
            template <class... Args> void emplace(Args&&... args);

    };

如果你忽略了构造函数，赋值操作和swap操作，那么剩下的接口就可以分为三类：

1. 查询队列状态，比如 `empty()` 和 `size()`
2. 查询队列元素，比如 `front()` 和 `back()`
3. 修改队列，比如 `push()` , `pop()` , `emplace()` 

这些就和之前的stack例子一样。
类似的，你也需要将 `front()` 和 `pop()` 结合起来作为一个操作，就像在stack中，你也需要结合 `top()` 和 `pop()` 。
Listing 4.1 中的代码有一些细微的差别，当使用一个队列来实现线程间传递数据时，
接收数据的线程常常需要等待数据。

我们为 `pop()` 添加两个变种：

1. `try_pop()` : 尝试从队列中pop一个数据，并且不管成功与否，都会立刻返回
2. `wait_and_pop()` : 等待，直到有数据为止

相关的接口如下

.. code-block:: c++
    :linenos:
    
    #include <memory>

    template<typename T>
    class threadsafe_queue
    {
    public:
        threadsafe_queue();
        threadsafe_queue(const threadsafe_queue&);
        threadsafe_queue& operator=(
                const threadsafe_queue&) = delete;

        void push(T new_value);

        bool try_pop(T& value);
        std::shared_ptr<T> try_pop();

        void wait_and_pop(T& value);
        std::shared_ptr<T> wait_and_pop();

        bool empty() const;
    };

类似于前面stack的实现，对于 `try_pop` 可以有两个重载：

1. 使用引用获得结果，并且可以将pop的状态用一个bool返回，如果获得数据，可以返回true，否则返回false。
2. 直接用指针返回数据，有数据，则直接返回指针，否则返回NULL.

Listing 4.3中的具体接口如下：

.. code-block:: c++
    :linenos:

    #include <queue>
    #include <mutex>
    #include <condition_variable>

    template<typename T>
    class threadsafe_queue
    {
    private:
        std::mutex mut;
        std::queue<T> data_queue;
        std::condition_variable data_cond;
    public:
        void push(T new_value)
        {
            std::lock_guard<std::mutex> lk(mut);
            data_queue.push(new_value);
            data_cond.notify_one();
        }

        void wait_and_pop(T& value)
        {
            std::unique_lock<std::mutex> lk(mut);
            data_cond.wait(lk, [this] {return !data_queue.empty();});
            value = data_queue.front();
            data_queue.pop();
        }

        threadsafe_queue<data_chunk> data_queue;

        void data_preparation_thread()
        {
            while(more_data_to_prepare())
            {
                data_chunk const data = prepare_data();
                data_queue.push(data);
            }
        }

        void data_processing_thread()
        {
            while(true)
            {
                data_chunk data;
                data_queue.wait_and_pop(data);
                process(data);
                if(is_last_chunk(data)) break;  // 线程退出条件
            }
        }
    };

现在需要的互斥量和条件变量都被threadsafe_queue内置了，所以，使用threadsafe_queue不需要例外的协同设置。

**Listing 4.5 Full class definition for a thread-safe queue using condition variables**

.. code-block:: c++
    :linenos:

    #include <queue>
    #include <memory>
    #include <mutex>
    #include <condition_variable>

    template<typename T>
    class threadsafe_queue
    {
    private:
        mutable std::mutex mut; // mutable 允许在const函数中修改类状态
        std::queue<T> data_queue;
        std::condition_variable data_cond;
    public:
        threadsafe_queue() {}
        threadsafe_queue(threadsafe_queue const& other)
        {
            std::lock_guard<std::mutex> lk(other.mut);
            data_queue = other.data_queue;
        }
        
        void push(T new_value)
        {
            std::lock_guard<std::mutex> lk(mut);
            data_queue.push(new_value);
            data_cond.notify_one();
        }

        void wait_and_pop(T& value)
        {
            std::unique_lock<std::mutex> lk(mut);
            data_cond.wait(lk, [this]{return !data_queue.empty();});
            value = data_queue.front();
            data_queue.pop();
        }

        std::shared_ptr<T> wait_and_pop()
        {
            std::unique_lock<std::mutex> lk(mut);
            data_cond.wait(lk, [this]{return !data_queue.empty();});
            std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
            data_queue.pop();
            return res;
        }

        bool try_pop(T& value)
        {
            std::lock_guard<std::mutex> lk(mut);
            if(data_queue.empty())
                return false;
            value = data_queue.front();
            data_queue.pop();
            return true;
        }

        std::shared_ptr<T> try_pop()
        {
            std::lock_guard<std::mutex> lk(mut);
            if(data_queue.empty())
                return std::shared_ptr<T>();
            std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
            data_queue.pop();
            return res;
        }

        bool empty() const  // 这里会用到mutable 的mutex
        {
            std::lock_guard<std::mutex> lk(mut);
            return data_queue.empty();
        }
    };

这里注意 `empty()` 函数，被标记为 `const` ，但是因为 `mut` 是 `mutable` 的，所以可以在构造函数和 `empty()` 中可以被修改。

条件变量在有多个线程在等待同一个事件的场景下也有用。
如果多个线程来协同完成同一个任务。
当一个新数据准备完毕，就调用 `notify_one()` 就会通知一个等候的线程来判断wait条件。
具体哪个线程被通知并不能确定，也可能所有的线程都正在处理数据，并没有线程在wait。

另外一种可能的情况是，多个线程都在等待同一个事件，而且它们全部都需要回复。
比如共享数据被初始化完毕，然后所有处理线程都需要被唤醒(尽管前面有更好的解决方式)。
或者多个线程需要等待共享数据的更新，比如一个定期的初始化。
在这些情况下，准备数据的线程可以调用 `notify_all()` 来唤醒所有的线程来检查自己在准备的条件是否已经成熟。

如果等候线程只等一次，当条件是 `true` ，那么它就不会再等待这个条件变量了，
那么一个条件变量也许不是最好的选择了。
在这个场景下， 一个 `future` 也许更加适合。

Waiting for one-off events with futures
------------------------------------------
假设你坐飞机去度假。 
一旦你来到机场，在完成了各种手续之后，你坐在座位上等待登机的通知。
你也许有很多方式来打发等待的时间，比如读书看报玩手机啥的，
但是最重要的是，你正在等待一件事：登机的通知。
这个通知是未来一次性的-- 下次你在来机场的时候，你会等待另外一架飞机。

C++ 标准库用一个称作 `future` 的来建模这类一次性的事件。
如果一个线程需要等待一个一次性事件，它以某种方式取得一个代表这个事件的 `future` 。
这个线程可以在未来的执行其他任务时会周期性地短时间等待来查看事件是否已经发生，
或者，它正在做另外一件事情，直到需要一个事件发生之后，它才继续进行。
一个 `future` 可以关联到数据。一旦其关联的事件发生了，就不能重新设置(reset)了。

在C++标准库中有两类 `feature` ，都被定义在 `<future>` 头文件中：
 
1. `unique feature` (std::future<>)
2. `shared futures` (std::shared_future<>)

`std::future` 的一个实例会独占一个事件，而 `std::shared_future` 可以将多个feature联系到同一个事件。

一次性事件中，最基础的就是在后台运行的计算返回结果。

Returning values from background tasks
-----------------------------------------
假设你有个长时间的计算任务，但是当前不需要结果。
你可以启动一个线程在后台执行计算任务，并且在前台线程需要结果的时候返回结果。
在这种场景下， `std::async` 函数模板就能发挥作用了。

你可以使用 `std::async` 来启动一个异步的任务，这个任务的结果你目前暂时不需要，但是在将来的某个时刻需要。
不同于给你返回一个 `std::thread` 对象，然后让你等待， `std::async` 会返回一个 `std::future` 对象，它会持有函数的返回结果。 

当你需要结果的时候，可以调用 `get()` ，那么当前线程会被阻塞，直到 `future` 的结果执行完毕了。

下面是一个例子：

**Listing 4.6 Using std::future to get the return value of an asynchronous task**

.. code-block:: c++
    :linenos:
    
    #include <future>
    #include <iostream>

    int find_the_answer_to_ltuae(); // 一个长时的计算任务 结果暂时不需要
    void do_other_stuff();

    int main()
    {
        std::future<int> the_answer<std::async(find_the_answer_to_ltuae);   // 开启异步线程计算
        do_other_stuff();
        std::cout<<"The answer is " << the_answer.get() << std::endl;   // 现在申请结果 
        return 0;
    }

和 `std::thread` 一样， `std::async` 允许你通过添加额外参数的方式为函数传递参数。
如果第一个参数是一个成员函数的指针，第二个参数提供了对象（直接提供，或者指针，或者 `std::ref` 封装）
，那么后续的参数将传递给对应的成员函数。
否则，第二个及后续的参数都会被传递给对应的函数。
就像 `std::thread` ，如果参数是rvalue，那么将会采用move操作传入。
这使得只支持move的参数的传递成为可能。

**Listing 4.7 Passing arguments to a function with std::async**

.. code-block:: c++
    :linenos:

    #include <string>
    #include <future>

    struct X
    {
        void foo(int, std::string const&);
        std::string bar(std::string const&);
    };

    X x;
    auto f1 = std::async(&X::foo, &x, 42, "hello"); // 调用 x.foo("hello")
    auto f2 = std::async(&X::bar, x, "goodbay");    // tmpx 是x复制而来
    struct Y
    {
        double operator() (double);
    };
    Y y;
    auto f3 = std::async(Y(), 3.141);   // tmpy 是Y() 通过move操作复制而来
    auto f4 = std::async(std::ref(y), 2.718);   // 直接调用 y(2.718)
    class move_only
    {
    public:
        move_only();
        move_only(move_only&&);
        move_only(move_only const&) = delete;
        move_only& operator=(move_only&&);
        move_only& operator=(move_only const&) = delete;
        void operator() ();
    }
    auto f5 = std::async(move_only());  // move_only的 std::move复制

默认情况下， `std::async` 是否会启动一个新线程取决于具体实现，或者当future在被等待时，任务是否是异步运行的。
在很多情况下，这就是你所需要的，但是你可以通过给 `std::async` 传递额外的参数来定制其运行的方式。

* `std::launch::deferred` 来指定函数调用延迟到调用 `wait()` 或者 `get()` 时
* `std::launch::async` 来指定函数以自己的线程异步运行
* `std::launch::deferred | std::launch::async` 来指定按默认方式运行

比如下面的例子：

.. code-block:: c++
    :linenos:

    auto f6 = std::async(std::launch::async, Y(), 1.2); // 开启新线程异步运行
    auto f7 = std::async(std::launch::deferred, baz, std::ref(x));  // 延迟云溪功能
    auto f8 = std::async(
            std::launch::deferred | std::launch::async,
            baz, std::ref(x));  // 默认方式运行
    auto f9 = std::async(baz, std::ref(x));
    f7.wait();  // 现在调用并执行被延迟的函数

Associating a task with a future
----------------------------------
`std::packaged_task<>` 将一个future绑定到一个函数或者可执行对象上。
当 `std::packaged_task<>` 对象被调用了，它会调用对应的函数或者可执行对象，然后将future设置为true，将结果存储起来。

`std::packaged_task<>` 的模板参数是一个函数签名，就像 `void()` 对应一个无传参无返回的函数，
或者 `int(std::string&, double*)` 对应一个两个传参返回int的函数。
当你构造了一个 `std::packaged_task` ，你必须传入一个函数或者可调用对象来接受具体类型的参数，但返回值的类型可以通过隐变化，比如int到float型。

特定函数签名的返回类型指定了 `std::future<>` 从 `get_future()` 成员函数返回的类型。
比如，一个 `std::packaged_task<std::string(std::vector<char>*, int)>` 局部的类的定义如下：

**Listing 4.8 Partial class definition for a specialization of std::packaged_task**

.. code-block:: c++
    :linenos:

    template<>
    class packaged_task<std::string(std::vector<char>*, int)>
    {
    public:
        template<typename Callable>
        explicit packaged_task(Callable&& f);
        std::future<std::string> get_future();
        void operator() (std::vector<char>*, int);
    };

`std::packaged_task` 对象可调用的对象，
而且可以被 `std::function` 作为可调用对象就行封装，或者被直接调用。
当 `std::packaged_task` 被作为函数对象调用了，那参数就会被传递给其包含的函数中，然后返回值被存储为 `std::future` 对象中的异步结果，future对象可以通过 `get_futre()` 取得。
你因此可以将一个任务封装到 `std::packaged_task` 对象中。
当你需要其结果时，你可以等待future准备完毕。 
下面的是实际的例子：

Parsing tasks between threads
******************************
许多GUI框架都需要从某个特定的线程来更新GUI，
所以，如果另外一个线程需要更新GUI，它必须向正确的线程发送一个消息。
`std::packaged_task` 提供了一种方式来避免为每个线程或者每种GUI操作都需要一种自定义的消息。

**Listing4.9 Running code on a GUI thread using std::packaged_task**

.. code-block:: c++
    :linenos:

    #include <deque>
    #include <mutex>
    #include <futurea>
    #include <thread>
    #include <utility>

    std::mutex m;
    std::deque<std::packaged_task<void()> > tasks;

    bool gui_shutdown_message_received();
    void get_and_process_gui_message();

    void gui_thread()
    {
        while(!gui_shutdown_message_received())
        {
            get_and_process_gui_message();
            std::packaged_task<void()> task;
            {
                std::lock_guard<std::mutex> lk(m);
                if(tasks.empty()) continue;
                task = std::move(tasks.front());
                tasks.pop_front();
            }
            task();
        }
    }

    std::thread gui_bg_thread(gui_thread);

    // 以任务为单位提交 而不是以消息为单位
    // 这样省去了接收message然后调用对应操作的麻烦
    template<typename Func>
    std::future<void> post_task_for_gui_thread(Func f)
    {
        std::packaged_task<void()> task(f);
        std::future<void> res = task.get_future();
        std::lock_guard<std::mutex> lk(m);
        tasks.push_back(std::move(task));
        return res;
    }

这个例子使用 `std::packaged_task<void()>` 来封装任务，具体的操作可以是不接收参数和返回数据的函数或可调用对象。

Making (std::)promises
**************************
当你有一个需要处理大量网络连接的应用，
那么用单独的线程来处理每个连接比较简单，因为这样可以使网络交互更加容易实现。
这个模式在连接比较少的时候比较适用，
但是当连接数目足够巨大时，为每个连接而分配的一个线程会导致总线程数巨大，而极大地耗费系统资源。
因此，常规的方法就是，分配固定数目的线程（也可能只有一个），而每个线程都需要处理多个连接。

想象这多个线程中的一个线程的工作机制。
数据包会从多个连接传过来，然后以一种随机的顺序被处理，
类似地，数据包在发送时，也会在队列中排列起来。
在很多情况中，应用的其他部分都会等待一个特定的网络连接成功发出或者接收数据。

`std::promise<T>` 提供了多种设置值的方法，这个值可以通过 `std::future<T>` 对象读取。
一个 `std::promise/std::future` 对可以提供这种机制；
等待线程可能在这个future上被阻塞，而提供数据的线程可以用promise来设置对应的数据，并且将future状态设置为完成。

就像 `std::packaged_task` ，你可以通过在 `std::promise` 上调用 `get_future()` 得到绑定的 `std::futrue` 对象.
当promise的值被设置了，那么对应的future的状态就会被设为完成，并且可以返回值。
如果你销毁一个没有设置值的 `std::promise` ，那么会得到一个异常。

Listing 4.10 展示了线程处理连接的例子。
在这个例子中，你使用一个 `std::promise<bool>/std::future<bool>` 对来标记数据的传输成功与否的状态；
绑定到future的数据就是简单的成功/失败标记。

.. code-block:: c++
    :linenos:

    #include <future>
    
    void process_connections(connection_set& connections)
    {
        while(!done(connections))
        {
            for(connection_iterator
                    connection = connections.begin(), 
                    end = connections.end();
                connection != end;
                ++condition)
            {
                if(connection->has_incomming_data())
                {
                    data_packet data = connection->incoming();
                    std::promise<payload_type>& p
                        = connection->get_promise(data.id);
                    p.set_value(data.payload);
                }
                if(connection->has_outgoing_data())
                {
                    outgoing_packet data = 
                        connection->top_of_outgoing_queue();
                    connection->send(data.payload);
                    data.promise.set_value(true);   // 成功发出后 更新标记
                }
            }
        }
    }

