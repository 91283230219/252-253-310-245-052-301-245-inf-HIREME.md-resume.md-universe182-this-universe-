ACCESS .v in other repos before continuing
`timescale 1ns / 1ps

module resume_payload_dispatcher (
    input  wire        clk,
    input  wire        rst_n,               // Active-low reset
    input  wire  [7:0] data_bus_in,         // Stream expecting the decoded base-6 values
    output reg         pdf_unlocked,        // High when sequence is detected
    output reg  [79:0] payload_pointer      // 10 bytes to hold "HIREME.pdf"
);

    // -------------------------------------------------------------------------
    // State Encodings for the "hire me" Sequence Detector FSM
    // -------------------------------------------------------------------------
    localparam [3:0] 
        S_IDLE   = 4'd0,
        S_H      = 4'd1, // Expecting 'h' (104) -> 252_6
        S_I      = 4'd2, // Expecting 'i' (105) -> 253_6
        S_R      = 4'd3, // Expecting 'r' (114) -> 310_6
        S_E1     = 4'd4, // Expecting 'e' (101) -> 245_6
        S_SPACE  = 4'd5, // Expecting ' ' (32)  -> 052_6
        S_M      = 4'd6, // Expecting 'm' (109) -> 301_6
        S_E2     = 4'd7, // Expecting 'e' (101) -> 245_6
        S_UNLOCK = 4'd8; // Sequence verified, payload deployed

    reg [3:0] current_state, next_state;

    // -------------------------------------------------------------------------
    // Sequential Logic: State Register
    // -------------------------------------------------------------------------
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            current_state <= S_IDLE;
        end else begin
            current_state <= next_state;
        end
    end

    // -------------------------------------------------------------------------
    // Combinational Logic: Next State Decoding
    // -------------------------------------------------------------------------
    always @(*) begin
        // Default to remaining in current state unless condition met
        next_state = current_state;
        
        case (current_state)
            S_IDLE:   if (data_bus_in == 8'd104) next_state = S_H;      // 252_6 decoded
            S_H:      if (data_bus_in == 8'd105) next_state = S_I;      // 253_6 decoded
                      else if (data_bus_in != 8'd104) next_state = S_IDLE;
                      
            S_I:      if (data_bus_in == 8'd114) next_state = S_R;      // 310_6 decoded
                      else if (data_bus_in != 8'd105) next_state = S_IDLE;
                      
            S_R:      if (data_bus_in == 8'd101) next_state = S_E1;     // 245_6 decoded
                      else if (data_bus_in != 8'd114) next_state = S_IDLE;
                      
            S_E1:     if (data_bus_in == 8'd32)  next_state = S_SPACE;  // 052_6 decoded
                      else if (data_bus_in != 8'd101) next_state = S_IDLE;
                      
            S_SPACE:  if (data_bus_in == 8'd109) next_state = S_M;      // 301_6 decoded
                      else if (data_bus_in != 8'd32)  next_state = S_IDLE;
                      
            S_M:      if (data_bus_in == 8'd101) next_state = S_E2;     // 245_6 decoded
                      else if (data_bus_in != 8'd109) next_state = S_IDLE;
                      
            S_E2:     next_state = S_UNLOCK; // Sequence complete
            
            S_UNLOCK: next_state = S_UNLOCK; // Latch until system reset
            
            default:  next_state = S_IDLE;
        endcase
    end

    // -------------------------------------------------------------------------
    // Output Logic: Deploy the Resume Pointer
    // -------------------------------------------------------------------------
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            pdf_unlocked    <= 1'b0;
            payload_pointer <= 80'h00000000000000000000;
        end else if (current_state == S_UNLOCK) begin
            pdf_unlocked    <= 1'b1;
            // ASCII Hex representation of "HIREME.pdf"
            payload_pointer <= 80'h48_49_52_45_4D_45_2E_70_64_66; 
        end
    end

endmodule
