---
title: '2025.01.06-2025.01.12 学习内容'
date: 2025-01-06
permalink: /posts/2025/1/blog-post-1/
tags:
  - Arm
  - 超级优化
  - C++
  - 向量寄存器
---

添加对类似"q1"的向量寄存器之支持。同时补充会引起sandbox报错的指令。

## 1、支持q寄存器

### 1.1 Antlr语法

添加即可。

```c++
// 添加q
fragment REG_VEC_NAME: ('b' | 'h' | 's' | 'd' | 'v' | 'q') ('0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9' | '10' | '11' | '12' | '13' | '14' | '15' | '16' | '17' | '18' | '19' | '20' | '21' | '22' | '23' | '24' | '25' | '26' | '27' | '28' | '29' | '30');
```

### 1.2 语义分析

参见asmjit文档：[asmjit::a64::Vec Class Reference](https://asmjit.com/doc/classasmjit_1_1a64_1_1Vec.html)

只有b、d、s、h、v，并没有q，所以这里的处理办法是将q完全等价于v处理。

在`reg_map`中添加一行处理q的代码：

```c++
const unordered_map<string, Reg> reg_map = []() {
    unordered_map<string, Reg> m;
    for (int i = 0; i < 32; ++i) {
        m["w" + to_string(i)] = GpW(i);
        // m["r" + to_string(i)] = GpW(i); // 'r' 其实是 AArch32
        // 里面的，这里我们就不支持了
        m["x" + to_string(i)] = GpX(i);

        // SIMD寄存器
        m["b" + to_string(i)] = VecB(i);
        m["h" + to_string(i)] = VecH(i);
        m["s" + to_string(i)] = VecS(i);
        m["d" + to_string(i)] = VecD(i);
        m["v" + to_string(i)] = VecV(i);
        // 添加这一行
        m["q" + to_string(i)] = VecV(i);
    }
    m["wzr"] = wzr;
    m["xzr"] = xzr;
    m["sp"] = sp;
    return m;
}();
```

## 2、引起报错的指令

部分指令会引起沙箱报错，结合之前工作进行了测试，这些指令的一些例子罗列如下：

```assembly
ld1 {v0.8b}, [x0]
ld2 {v0.8b, v1.8b}, [x0]
ld3 {v0.8b, v1.8b, v2.8b}, [x0]
ld4 {v0.8b, v1.8b, v2.8b, v3.8b}, [x0]

ld1r {v0.8b}, [x0]
ld2r {v0.8b, v1.8b}, [x0]
ld3r {v0.8b, v1.8b, v2.8b}, [x0]
ld4r {v0.8b, v1.8b, v2.8b, v3.8b}, [x0]

st1 {v0.8b}, [x0]
st2 {v0.8b, v1.8b}, [x0]
st3 {v0.8b, v1.8b, v2.8b}, [x0]
st4 {v0.8b, v1.8b, v2.8b, v3.8b}, [x0]
```

目前可以确定的是，有且只有这些指令会引起报错，其他指令均不会报错。这些指令的特点是只能作为向量指令使用；其他不会引起报错的指令，如ldp，既可以作为标量指令（此时对应asmjit中InstId为"kIdLdp"），也可以作为向量指令（此时对应asmjit中InstId为"kIdLdp_v"）使用。

故补充test_excutor函数如下：

```c++
TEST_F(ExecutorTestFixture, St1)
{
  /* 目前可以确定的是，只有 ld1..ld4/ld1r...ld4r/st1..st4 这12条指令会引起报错，这些指令的特点是只能作为向量指令使用。
   * 其他不会引起报错的指令，如ldp，既可以作为标量指令（此时对应asmjit中InstId为"kIdLdp"）使用，也可以作为向量指令（此时对应asmjit中InstId为"kIdLdp_v"）使用。
   */
  Config *config = Config::getInstance();
  config->def_in_registers.insert({x0, v0});
  config->live_out_registers.insert({x0, v0});
  config->available_registers.insert({x0, v0});
  /*
    st1 {v0.8b}, [x0]
    将 v0 寄存器的 8 个字节元素存储到 `x0` 指向的内存中的位置。
  */
  BasicBlock bb1({
      TInstNodeHandler(Inst::kIdSt1_v, { VecV(0).b8(), Mem(GpX(0))}),
  });
  Program p;
  p.addBlock(make_shared<BasicBlock>(bb1));
  // 生成输入测例
  CPUState input = gen_rand_ptr();
  cout << "Input: " << input << endl;
  auto [exit_code, res] = run(p, input);
  ASSERT_TRUE(exit_code == NORMAL);
}
```

