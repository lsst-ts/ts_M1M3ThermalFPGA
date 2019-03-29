Usage:
1. Generate a target-scoped FIFO of block memory for internal health and status usage. Specify the data type as the HealthAndStatusUpdate cluster. Specify never arbitrate for read and always arbitrate for write. When specifying the number of elements give some room based on the number of functions that will be providing data to the health and status interface.
2. Generate a host to target - DMA FIFO for query communications. Specify the data type as U64. Specify never arbitrate for read. When specifying the number of elements give some room for multiple requests but not too many, a value of 4 should be more than enough.
3. Generate a target to host - DMA FIFO for query responses. Specify the data type as U32. Specify never arbitrate for read. When specifying the number of elements make sure to account for the total size of the health and status memory. If your health and status memory block is 128 and your response FIFO is only 32 then when you request for all health and status data you will run into issues. A good rule of thumb is to use 2x the block memory size.
4. Generate a memory block for health and status storage in the FPGA. Specify the data type as U32. When specifying the number of elements make sure to take into account all of the address space required for the application.
5. Drop the HealthAndStatus.vi into your main FPGA vi, link up the created FIFO's and memory blocks.
5. Use the internal health and status FIFO for creating update requests which are processed by the ProcessHealthAndStatusUpdates.vi. It is recommended to create a helper VI for putting the health and status update data onto the FIFO. One is provided in the library, it is called UpdateHealthAndStatus.vi.

Notes:
- Don't try to put something into the health and status that updates every 5 ticks, that will easily eat up all of the processing time and your health and status data will get lost as FIFOs overflow. Items that update once every 10 microseconds are fine as the update loop for health and status executes much faster than that.
- If using a single cycle loop you will need to create an internal FIFO for pushing health and status updates to, then create another loop to pull out of that internal FIFO and pushing into the internal health and status FIFO. 

