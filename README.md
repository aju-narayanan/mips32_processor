# mips32_processor
module pipe_mips32(clk1,clk2

    );
    input clk1,clk2;
    reg [31:0]pc,if_id_ir,if_id_npc;
    reg [31:0]id_ex_ir,id_ex_npc,id_ex_a,id_ex_b,id_ex_imm;
    reg [2:0]id_ex_type,ex_mem_type,mem_wb_type;
    reg[31:0]ex_mem_ir,ex_mem_aluout,ex_mem_b;
    reg ex_mem_cond;
    reg [31:0]mem_wb_ir,mem_wb_aluout,mem_wb_lmd;
    
    reg [31:0]Reg[0:31];
    reg [31:0]Mem[0:1023];
    //reg [31:0]Mem_data[0:1023];
    
    parameter beqz=6'b001110,bneqz=6'b001101,add=6'b000000,sub=6'b000001,
              and_=6'b000010,or_=6'b000011,slt=6'b000100,mul=6'b000101,
              hlt=6'b111111,lw=6'b001000,sw=6'b001001,addi=6'b001010,
              subi=6'b001011,slti=6'b001100; 
    
    parameter rr_alu=3'b000,rm_alu=3'b001,load=3'b010,store=3'b011,branch=3'b100,halt=3'b101;
    
    reg halted;
    reg taken_branch;
    
    
    
    
    
    always@(posedge clk1)
    if(halted==0)
    begin
    if (((ex_mem_ir[31:26]==beqz)&&(ex_mem_cond==1))||((ex_mem_ir[31:26]==bneqz)&&(ex_mem_cond==0)))//branch taken
    begin
    if_id_ir      <=#2Mem[ex_mem_aluout];
    taken_branch  <=#21'b1;
    if_id_npc     <=#ex_mem_aluout+1;
    pc            <=#2ex_mem_aluout+1;
    end
    
    else
    begin
    if_id_ir   <=#2Mem[pc];
    if_id_npc  <=#2pc+1;
    pc         <=#2pc+1;
    end
    end
    
    always@(posedge clk2)
    if(halted==0)
    begin
    if(if_id_ir[25:21]==5'b00000) id_ex_a<=0;
    else id_ex_a <=#2Reg[if_id_ir[25:21]];//rs
    
    if(if_id_ir[20:16]==5'b00000) id_ex_b<=0;
    else id_ex_b <=#2Reg[if_id_ir[20:16]];//rt
    
    id_ex_npc=#2if_id_npc;
    id_ex_ir=#2if_id_ir;
    id_ex_imm=#2{{16{if_id_ir[15]}},{if_id_ir[15:0]}};
    
    case(if_id_ir[31:26])
    add,sub,and_,or_,slt,mul   :id_ex_type  <=#2rr_alu;
    addi,subi,slti   :id_ex_type  <=#2rm_alu;
    lw   :id_ex_type  <=#2load;
    sw   :id_ex_type  <=#2store;
    bneqz,beqz   :id_ex_type  <=#2branch;
    hlt   :id_ex_type  <=#2halt;
    default   :id_ex_type  <=#2halt;    
   endcase
   end
   
   always@(posedge clk1)
   if(halted==0)
   begin
   ex_mem_type<=#2id_ex_type;
   ex_mem_ir<=#2id_ex_ir;
   taken_branch<=#2 0;
   
   case(id_ex_type)
   rr_alu: begin
    case(id_ex_ir[31:26])
    add:ex_mem_aluout <= #2 id_ex_a+id_ex_b;
    sub:ex_mem_aluout <= #2 id_ex_a-id_ex_b;
    and_:ex_mem_aluout <= #2 id_ex_a&id_ex_b;
    or_:ex_mem_aluout <= #2 id_ex_a|id_ex_b;
    slt:ex_mem_aluout <= #2 id_ex_a<id_ex_b;
    mul:ex_mem_aluout <= #2 id_ex_a*id_ex_b;
    default:ex_mem_aluout <= #2 32'hxxxxxxxx;
    endcase
    end
    rm_alu: begin
             case(id_ex_ir[31:26])
             addi:ex_mem_aluout <= #2 id_ex_a+id_ex_imm;
             subi:ex_mem_aluout <= #2 id_ex_a-id_ex_imm;
             slti:ex_mem_aluout <= #2 id_ex_a<id_ex_imm;
             default:ex_mem_aluout <= #2 32'hxxxxxxxx;
             endcase
             end
             
    load,store: begin
            ex_mem_aluout<=#2 id_ex_a+id_ex_imm;
            ex_mem_b <=#2 id_ex_b;
            end
    branch: begin
            ex_mem_aluout<=#2 id_ex_npc+id_ex_imm;
            ex_mem_cond <=#2 (id_ex_a==0);
            end        
     endcase
     end



always@(posedge clk2)
if(halted==0)
begin
mem_wb_type <=#2 ex_mem_type;
mem_wb_ir <=#2 ex_mem_ir;

case(ex_mem_type)
rr_alu,rm_alu : mem_wb_aluout <=#2 ex_mem_aluout;
load : mem_wb_lmd <=#2 Mem[ex_mem_aluout];
store : if(taken_branch==0) Mem[ex_mem_aluout] <= #2 ex_mem_b;
endcase
end

always@(posedge clk1)
begin
if(taken_branch==0)
case (mem_wb_type)
rr_alu : Reg[mem_wb_ir[15:11]] <= #2 mem_wb_aluout;
rm_alu : Reg[mem_wb_ir[20:16]] <= #2 mem_wb_aluout;
load : Reg[mem_wb_ir[20:16]]    <= #2 mem_wb_lmd;
halt : halted <= #2 1'b1;
endcase
end
endmodule
