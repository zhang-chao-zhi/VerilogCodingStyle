##########
2 高级语法
##########

2.1 参数化电路设计规范
**********************


a) 不显示声明 **generate** 和 **endgenerate**。不声明在标准中允许，且没有任何影响。
b) 给必要的生成块 **添加block name** ，带有block name的块需要添加对应的 **ending name**。若在生成语句中调用module或定义信号，该module或信号通过block name访问。若没有显示定义block name，仿真器或综合器会自动生成。**在使用生成块时，尽量给所有的生成块命名。**
c) 互斥的block可以使用 **相同的block name**。例如：

   .. code-block:: verilog
   
     if(...) begin : BlkName 
       ... 
     end else begin : BlkName
       ... 
     end 
   
d) 所有在生成语句中使用的所有变量都是 **固定值** (参数 或 宏)，不是电路中的信号。
e) 条件电路生成使用宏定义实现，用以区分电路中信号的 if 判断。

   .. code-block:: verilog

     `define gen_if   if
     `define gen_elif else if
     `define gen_else else

     `gen_if(P_PARAM_A == 1) begin : dat
       assign dat1 = xx;
     end `gen_else begin: dat
       assign dat1 = yy;
     end

     always_comb begin
       `gen_if(P_PARAM_A == 1) begin
         dat2 = xx;
       end `gen_else begin
         dat2 = yy;
       end
     end

f) 生成块中for循环写法：**for(genvar i=0; i<xx; i++)**
g) always中for循环写法：**for(int i=0; i<xx; i++)**
h) for循环的边界判断尽量使用系统函数根据信号进行自动推断。循环变量自加可以使用 **‘i++’** 计算符。
i) always中如果需要遍历一个向量内的所有信号，使用foreach循环实现：**foreach(dat[i])**

  .. code-block:: verilog

    // Define signal outside the loop generate block is recommended.
    logic [P_WIDTH-1:0] dat1;
    // It is recommended to use $bits(dat1) instead of P_WIDTH.
    for(genvar i=0;i<$bits(dat1);i++) begin : dat1_Gen
      assign dat1[i] = xx[i];
    end : dat1_Gen

    // Use P_WIDTH is also allowed.
    for(genvar i=0;i<P_WIDTH;i++) begin : dat2Block
      wire dat2 = xx[i];  // The signal dat2 can only be accessed by block name: dat2Block.
    end : dat2Block_Gen

    logic [P_WIDTH-1:0] dat3,dat4;
    always_comb begin
      for(int i=0;i<$bits(dat3);i++) begin
        dat3[i] = xx[i];
      end
    end

    always_comb begin
      foreach(iDatA[i]) begin
        dat4[i] = xx[i];
      end
    end

2.2 struct 用法规范
********************

  .. code-block:: verilog

    // Define a struct signal directly.
    struct packed{
      logic [7:0] dat;
    }dat1St;
    
    // Define a struct type and define a struct signal by the new type.
    typedef struct packed{
      logic [7:0] dat;
    }type_Demo2St;
    type_DemoAaSt dat2St;
    
    // Parameterized struct definition by macro.
    `define typedef_DemoSt(width) \
      typedef struct packed{\
        logic [width-1:0] dat;\
      }
    `define type_DemoSt(width) \
      struct packed{\
        logic [width-1:0] dat;\
      }

    // Define struct type and struct variable by macro.
    `typedef_DemoSt(8) type_Demo3St;
    type_Demo3St dat3St;
    `type_DemoSt(8) dat4St;
    
    // Type convert by casting operating.
    assign dat3St = dat3St'(dat4St);

a) 同方向有相关性信号，推荐使用struct定义。
b) 结构体定义必须使用packed形式。
c) 直接使用struct定义在不同位置的变量会被EDA工具认为是两个不同变量。当需要在多处定义相同struct时，使用typedef形式定义类型，使用 **type_** 作为前缀，类型名用 **大驼峰** 命名法，结尾用 **St** 作为后缀。
d) struct定义的变量用 **小驼峰** 命名法，**St** 作为后缀。
e) 使用宏实现参数化struct定义，建议同时定义 typedef 和 非typedef 两种方式。两种宏分别以：**typedef_** 和 **type_** 作为前缀，使用 **大驼峰** 命名法，**St** 作为后缀。(SystemVerilog标准中使用virtual class实现参数化struct定义，该语法尚未被部分EDA工具支持。)
f) struct 可以使用 **'( )** 操作符。
g) union定义方式与struct相同，变量后缀为 **Un** 。

2.3 package 用法规范
*********************

  .. code-block:: verilog

    package BasicPkg;
      parameter P_A = 1;

      function automatic logic [3:0] DatAnd(input [3:0] in1,in2)
        return in1 & in2;
      endfunction
    endpackage

a) 有相关性的信号、参数、数据类型、函数可以集合在一起定义在一个package内。
b) package以 **大驼峰** 方式命名，以 **Pkg** 作为名称结尾。
c) package内的定义都不支持参数化。(SystemVerilog标准中尚不支持)
d) package中定义的function必须 **包含automatic** 声明。

2.4 interface 用法规范 
**********************

  .. code-block:: verilog

    interface TestItf
    #(P_A
    );

      logic [3:0] datOh;  // All signal defined in 'logic'.
      logic [1:0] dat;
      logic       datOh0,datOh1,datOh2,datOh3;
      typedef struct packed{logic dat1;logic [1:0] dat2;} type_DataSt;
      assign datOh0 = datOh[0];
      assign datOh1 = datOh[1];   // Only bit selection/extension is allowed.
      assign datOh2 = datOh[2];
      assign datOh3 = datOh[3];

      function automatic void Codec;  // 'automatic' is necessary.
        dat = {(datOh3|datOh2),(datOh3|datOh1)};
      endfunction
      function automatic logic BiggerThan1;
        return {(dat > 2'd1),dat};
      endfunction

      modport datOhOut(output datOh);
      modport datIn(input dat, import BiggerThan1); // import function in modport.
      modport Unit(input datOh0,datOh1,datOh2,datOh3, output dat, import Codec);

    endinterface : TestItf

    module TestItfUnit
    (
      TestItf.Unit bDatIf
    );
      bDatIf.Codec();
    endmodule : TestItfUnit

    module ModuleBb
    (
      TestItf.datIn iDatIfIf,
      output logic oResult
    );
      typedef iDatIf.type_DataSt type_DatSt;  // Use typedef in interface.
      type_DatSt dataSt;
      assign dataSt = iDatIf.BiggerThan1();   // Use function in interface.
      assign oResult = dataSt.dat1;
    endmodule : ModuleBb

a) interface名称定义使用 **Itf** 作为后缀，信号定义使用 **If** 作为后缀。内部信号使用 **logic** 或 **struct** 定义。
b) 用于模块间互联的interface中，值允许存在 **位选择、位截取、位扩展** 逻辑电路，不能存在任何会生成具体器件的逻辑电路。

  - interface中实现的电路逻辑在综合后会直接出现在例化interface的module中，这种写法不利于综合、后端流程。因此不允许直接在interface中实现具体电路。
  - 位选择、位截取、位扩展逻辑并不存在实际电路，只是改变连接关系，不影响其他流程。

c) 实现与interface相关性较高的逻辑，通过以下两种方式：

  - 单独实现一个module。

    设计一个单独的module，将interface作为接口，将逻辑放置在module内。对于复杂的电路实现，推荐这种方式。

  - 在interface中设计 **function** 或 **task**。
  
    1. 将需要实现的功能设计在interface的 **function** 或 **task** 中。
    2. 通过modport将 function 或 task 直接 **import** 到module中。
    3. 在module中直接调用。
    4. function 或 task 可以直接访问interface里的信号，不需要通过端口传递。
    5. 建议只使用function，不使用task。在function中不要放置带复杂的逻辑。

d) 可以在interface中typedef数据类型，通过interface将数据类型引入到module中。
e) 标准中允许在module中直接访问interface中的parameter，该功能目前尚未被EDA工具支持。(TODO:EDA工具更新后重新测试)
f) 减少在interface中的input信号数量，尤其是会参与计算的信号。在测试中遇到过相关EDA工具Bug。
g) interface在端口定义和信号连接时必须 **指定modport** 。否则综合会提示信号未使用warning。
h) 通过 **interface + modport + 参数化设计** 可以实现verilog可变端口数量。
i) 利用interface可以进行电路封装，将通用电路在interface内实现，可选电路使用function实现。在使用时，根据需要调用对应的function。在此情况下interface内可以包含实际电路。

TODO：在附录中给出各种复杂设计下的Demo。
