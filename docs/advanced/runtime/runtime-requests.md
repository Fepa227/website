---
title: Runtime requests
redirect_from:
  - /RuntimeRequests/
---

SIMD
====

Alinhamento
-----------

É importante manipular objetos SIMD em uma boa estruturas para que eles fiquem com padrão devidamente alinhados, caso contrário, sofrerá alguns problemas de desempenho, tais como a utilização de cargas desordenadas, causando 10x mais diferenças devido ao alinhamento aleatório.

ABI
---

Seria possível passar argumentos SIMD nos registradores SIMD na Intel?

=\> Isso é possível, mas desaconselhável até nós reescrevermos o alocador de registro.

Portas
------

Gostaria de ter LLVM / ARM suportado pelo SIMD (NEON), pois isso nos ajudaria no MonoTouch. Talvez tenhamos o mesmo em MonoJIT / ARM para outras plataformas como o Android.

Seria bom para apoiar PowerPC SIMD (AltiVec) para plataformas como PS3.

Processadores x86 mais novos (Intel Sandy Bridge e AMD Bulldozer) introduzem uma outra extensão SIMD chamada AVX, e seria bom o suporte para isso também.

Implementações de método
------------------------

Uma vez que muitas instruções SIMD existem em um conjunto de instruções especificas ou na extensão do conjunto de instruções, seria útil ter uma maneira de ter diferentes versões de um método para diferentes arquiteturas, por exemplo um para SSE1, outro para SSE2, e outro para NEON. Talvez isso poderia ser feito com um atributo e um nome de método sufixo codificado, por exemplo:

     [MonoMethodImpl(MethodImplOptions.ArchSpecific)] void Foo ()

em seguida, se o processador suporta SSE2 e o método:

     void Foo_Sse2 ()

existindo, seria usado em seu lugar.

Estruturas envolvidas em SIMD
-----------------------------

Quando são produzidos bons APIs, muitas vezes é útil envolver o SIMD com tipos intrínsecos de estruturas de API mais limpas e mais específicas para uma finalidade certa, por exemplo, um Quaternion que envolve um campo Vector4f. Infelizmente, o JIT atualmente gera um código horrível para esses casos, uma vez que não lida bem indiretamente, especialmente quando combinada com SSE intrinsecos.

Para exemplos de código gerados, consulte [https://bugzilla.novell.com/show_bug.cgi?id=662127](https://bugzilla.novell.com/show_bug.cgi?id=662127)

Referencias sobre sobrecargas em Mono.Simd
------------------------------------------

Fornece referências de sobrecargas para todos os métodos em Mono.SIMD, desde implementações não-intrínsecas, passando por grandes estruturas, é geralmente muito mais rápido do que passá-los por valores.

Ponto Flutuante SSE em x86
-------------------------

Devemos usar matemática de ponto flutuante SSE em x86, como fazemos em x86-64, em vez de usar o FPU x87, como fazemos agora.

Otimizando a indução
====================

ABC desabilitado
----------------

Adicionar um atributo (talvez MonoMethodImpl) para desativar limites de matriz em métodos específicos. Isso permitiria que ele seja desativado no código da biblioteca, mantendo-o no código do usuário. Obviamente, isso só seria permitido para métodos não-seguros.

Branch induzido
--------------

Adicionar JIT intrinsecospara branch induzido, para ajustar o código como usado no Mono.Simd.

Dados Prefetch
-------------

JIT intrinsics for data prefetch instructions. Useful combined with Mono.Simd.

Byref attributes
----------------

Add an attribute to be applied to struct method parameters to indicate that the JIT should pass them by reference, while retaining a by-value API in the CIL. This would allow byref args for operator overloads, and would remove the need to create byval and byref overloads for perf.

Optimization Level
------------------

Allow using MonoMethodImpl attribute to hint that a method is important and should be optimized more heavily, maybe even using LLVM.

Inliner
=======

Force Inline Attribute
----------------------

Perhaps we can steal one of the attributes in the MethodImpl to force inlining for certain methods. If not, perhaps we could add a new MonoMethodImpl attribute for our own JIT optimization control.

Not sure if Mono can inline any method, or if there are limitations on what we can inline, even when forced to inline. Apparently .NET 3.5 can't inline methods with struct parameters, which hurts perf of math vector APIs really badly - maybe this is somewhere we could do much better?

Intrinsics
----------

Would it be possible to inline certain common code patterns like List\<T\>.this [int idx]?

Is the inliner still limited in cases where there is a compare and branch code? Could this limitation be removed?

This would **really** improve inlining of common patterns such as properties with argument checking, since in many cases the JIT (or LLVM) could do dead code elimination of the argument checks. For example, in the case

``` csharp
a = new Foo ();
b.Property = a;
```

where b.Property is:

``` csharp
Foo Property {
   set {
      if (value == null) throw new ArgumentNullException ();
      x = value;
   }
}
```

then LLVM can do dead-code elimination on the "if value==null"

P/Invoke Inlining
=================

Would it be possible to have an attribute to P/Invoke that would flag "this is a simple method that should be treated as an internalcall, do not setup any expensive wrappers", like for methods that just call into C and are known to not throw exceptions and have a finite execution time (so we do not need to handle Thread.Interrupt there).

This would help us improve the speed of calling P/Invoke methods.

What is determined "safe" I am not sure, would love to figure out what we can do about this.

The concern is not as much the size of the generated wrappers, but the need to execute those wrappers.

Why the simple solution is not possible
---------------------------------------

icall wrappers are needed to be able to do stack walks too. Plus for handling async exceptions. i.e. if a thread gets a signal while it executes an icall, the icall wrapper will throw the ThreadAbortException or such when the icall returns.

To be able to do stack walks, we need to save some state before calling native code, to be able to handle async exceptions, we need to do a check after each native call.

Some of this could be inlined at the call site, but calling a native method will never be equivalent to 'call sin', it will always have some overhead.

What can be done
----------------

We need either the wrappers or the functionality they contain.

We could inline some parts of it, its tricky but doable, that would save the call+parameter passing overhead.

Its not worth it for common icalls like allocators, since they blow up the size of call sites, but might be worth for icalls which are called from 1 place.

ICALL performance
-----------------

Here is a benchmark to test icall performance:

``` bash
using System;
using System.Diagnostics;
 
public class Tests
{
    public static void Main (String[] args) {
        for (int i = 0; i < 10; ++i) {
            int k = (i + 1) / (i + 1);
        }
        var s = Stopwatch.StartNew ();
        int niter = 10000000;
        for (int i = 0; i < niter; ++i) {
            int k = (i + 1) / (i + 1);
        }
        TimeSpan t = s.Elapsed;
        Console.WriteLine (t.Milliseconds);
        Console.WriteLine ("" + (niter / t.Milliseconds) + " iterations per ms");
    }
}
```

On an NVIDIA TEGRA, this runs in:

-   normal case: 824ms.
-   normal case, calling mono_get_lmf_addr instead of an inline TLS get sequence: 873ms.
-   normal case, save_lmf=FALSE: 490ms.
-   normal case, create the LMF structure, but don't push/pop it: 590ms.
-   without any wrappers at all: 372ms.

Here is the assembly for the wrapper:

``` bash
   -> save registers and allocate stack frame
   0:   e1a0c00d        mov     ip, sp
   4:   e92d5ff0        push    {r4, r5, r6, r7, r8, r9, sl, fp, ip, lr}
   8:   e24dd030        sub     sp, sp, #48     ; 0x30
   -> save arguments to stack/registers
   c:   e58d0000        str     r0, [sp]
  10:   e1a07001        mov     r7, r1
   -> load lmf_addr TLS variable
  14:   ebfff322        bl      0xffffcca4
  18:   e590000c        ldr     r0, [r0, #12]
  -> create LMF structure on the stack
  1c:   e28d100c        add     r1, sp, #12
  20:   e5810004        str     r0, [r1, #4]
  24:   e5902000        ldr     r2, [r0]
  28:   e5812000        str     r2, [r1]
  2c:   e5801000        str     r1, [r0]
  30:   e581d00c        str     sp, [r1, #12]
  34:   e1a0200f        mov     r2, pc
  38:   e5812010        str     r2, [r1, #16]
  -> load arguments and make the call
  3c:   e59d0000        ldr     r0, [sp]
  40:   e1a01007        mov     r1, r7
  44:   ebfff31c        bl      0xffffccbc
  48:   e1a01000        mov     r1, r0
  -> load interruption flag
  4c:   e30000a8        movw    r0, #168        ; 0xa8
  50:   e340003c        movt    r0, #60 ; 0x3c
  54:   e5900000        ldr     r0, [r0]
  58:   e1a07001        mov     r7, r1
  -> check it, and branch to interruption code if needed
  5c:   e3500000        cmp     r0, #0
  60:   1a000006        bne     0x80
  -> load return value
  64:   e1a00007        mov     r0, r7
  -> restore LMF
  68:   e28d200c        add     r2, sp, #12
  6c:   e592c000        ldr     ip, [r2]
  70:   e592e004        ldr     lr, [r2, #4]
  74:   e58ec000        str     ip, [lr]
  -> pop stack frame and return
  78:   e282d030        add     sp, r2, #48     ; 0x30
  7c:   e8bd9f80        pop     {r7, r8, r9, sl, fp, ip, pc}
  -> interruption code
  80:   ebfffeac        bl      0xfffffb38
  84:   eafffff6        b       0x64
```

Support for NSString
====================

Wondering if we could add built-in knowledge to turn a Mono String into an Objective-C NSString in the same way that we do instrinsics for SIMD. The idea would be to just create a simple shell structure for NSString that is initialized to point to the UTF16 data in Mono's string.

GCC compiled a regular NSString constant as something like:

    void *classptr;
    intxx flags;
    void *ucs2data

The idea would be to have a mechanism that allocates this in the stack, and allows us to pass a pointer to this blob to the Objective-C code. In C# this would look like this:

    string a = "hello";
    Some_PInvoke_to_objective_C (Mono.Rutnime.AsNsString (a));

The above would do something like a stackalloc, fill in the class pointer, the flags and the dta pointer to point to Mono's string.

I don't see why this would need stackalloc or special JIT support. You should be able to just define a C# struct and fill the ucs2data field with a fixed expression. All this would happen in an autogenerated wrapper of the P/Invoke call. There would be no speed or memory advantage at doing this specially in the JIT.

