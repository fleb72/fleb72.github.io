---
title: "Abstraction Levels in Hardware Description Languages (HDL) - Part 1"
date: 2025-04-10
categories: [FPGA]
tags: [FPGA]
math: true
---

Hardware Description Languages (HDLs) such as VHDL or Verilog/SystemVerilog are not classic programming languages such as C, C++, Java, Python, etc. Even if procedural sequences are found in the syntax of these languages, it is no longer a question of producing sequences of instructions executed sequentially by a CPU.

[multiplexer](https://en.wikipedia.org/wiki/Multiplexer) (sometimes called a *data selector*) in SystemVerilog (an extension of the Verilog language).

### Definition of a multiplexer (abbreviated *MUX*)

![Multiplexer schematic](/assets/img/posts/2025-04-10-fpga-abstraction-level/mux.png)
*Multiplexer Schematic*

$$
s = a \cdot \overline{sel} + b \cdot sel
$$

### Structural description


![Logical diagram of the multiplexer](/assets/img/posts/2025-04-10-fpga-abstraction-level/mux_logic_cells.png)
*Logical diagram of the multiplexer*


```verilog
module mux21  // 2 inputs - 1 output multiplexer
   (
      output   logic s,
      input    logic a, b, sel      
   );
         
   logic q1, q2, sel_n; 

   not (sel_n, sel);
   and (q1, a, sel_n);  
   and (q2, b, sel);   
   or  (s, q1, q2);
  
endmodule
```


![Multiplexer schematic](/assets/img/posts/2025-04-10-fpga-abstraction-level/mux41.png)

