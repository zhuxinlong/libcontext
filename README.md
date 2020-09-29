# libcontext
A stripped-down, portable version of boost::context


```cpp
// https://github.com/twlostow/libcontext.git
/**
 * @biref 执行环境上下文
 */
typedef void*   fcontext_t;

/**
 * @biref 跳转到目标上下文
 * @param ofc 当前的上下文会保存到ofc中
 * @param nfc 跳转到的目标上下文
 * @param vp 如果是第一次跳转，作为函数参数传入，如果是调回到jump_fcontext，这个是返回值
 * @param preserve_fpu 是否复制FPU（浮点数寄存器）数据
 * @return 如果调回时的透传参数
 */
intptr_t LIBCONTEXT_CALL_CONVENTION jump_fcontext( fcontext_t * ofc, fcontext_t nfc,
                                               intptr_t vp, bool preserve_fpu = false);
/**
 * @biref 初始化执行环境上下文
 * @param sp 栈空间地址
 * @param size 栈空间的大小
 * @param fn 入口函数 参数为jump_fcontext填入的vp
 * @return 返回初始化完成后的执行环境上下文
 */
 
fcontext_t LIBCONTEXT_CALL_CONVENTION make_fcontext( void * sp, size_t size, void (* fn)( intptr_t));

```
## demo
```cpp
#include "libcontext.h" 
#include <array>
#include <functional> 
#include <iostream> 
class Coroutine
{
public:
    Coroutine() 
    {
        my_context = new fcontext_t();
        yield_context = new fcontext_t();
        *my_context = make_fcontext(
            stack.data() + stack.size(),
            stack.size(),
            Coroutine::dispatch
        );

        std::cout << "init context!\n";
    }
    virtual ~Coroutine() {
        delete my_context;
        delete yield_context;
    }

    void operator()()
    {
        std::cout << "yield_context ===> my_context \n";

        jump_fcontext(yield_context, *my_context, reinterpret_cast<intptr_t>(this));
    }

protected:
    void yield()
    {
        jump_fcontext(my_context, *yield_context, 0);
    }

    virtual void call() = 0;

private:
    static void dispatch(intptr_t coroutine_ptr)
    {
        Coroutine *coroutine = reinterpret_cast<Coroutine *>(coroutine_ptr);
        coroutine->call();
        while (true)
            coroutine->yield();
    }

private:
    fcontext_t* my_context;
    fcontext_t* yield_context; 
    std::array<intptr_t, 64*1024> stack;

};

struct A : public Coroutine
{
    void call()
    {
        std::cout << "A 1.\n";
        yield();
        std::cout << "A 2.\n";
        yield();
        std::cout << "A 3.\n";
    }
};

struct B : public Coroutine
{
    void call()
    {
        std::cout << "B 1.\n";
        yield();
        std::cout << "B 2.\n";
        yield(); 
    }
};
int main()
{ 
    A a;
    B b; 
    for (size_t i = 0; i < 10; ++i)
    {
        a();
        b(); 
    }
    std::cout << "done.\n";
}

//g++ -std=c++11 -Wall -g -m64 -I. test.cpp  ./libcontext/libcontext.cpp -o test-linux-gcc-x86-64 -I./libcontext
```
