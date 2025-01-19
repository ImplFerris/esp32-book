# Access Control

The tag includes access bits that enable access control for the data stored in the tag. This chapter will explore how these access bits function. This section might feel a bit overwhelming, so I'll try to make it as simple and easy to understand as possible.

<div class="alert-box alert-box-danger">
    <span class="icon"><i class="fa fa-flash"></i></span>
    <div class="alert-content">
        <b class="alert-title">Modifying Access Bits</b>
        <p>Be careful when writing the access bits, as incorrect values can make the sector unusable. </p>
    </div>
</div>


## Permissions
These are the fundamental permissions that will be used to define access conditions. The table explains each permission operation and specifies the blocks to which it is applicable: normal data blocks (read/write), value blocks, or sector trailers.

| **Operation** | **Description**                                            | **Applicable for Block Type**         |
|---------------|------------------------------------------------------------|-----------------------------------|
| **Read**      | Reads one memory block                                     | Read/Write, Value, Sector Trailer |
| **Write**     | Writes one memory block                                    | Read/Write, Value, Sector Trailer |
| **Increment** | Increments the contents of a block and stores the result in the internal Transfer Buffer | Value                             |
| **Decrement** | Decrements the contents of a block and stores the result in the internal Transfer Buffer | Value                             |
| **Restore**   | Reads the contents of a block into the internal Transfer Buffer | Value                             |
| **Transfer**  | Writes the contents of the internal Transfer Buffer to a block | Value, Read/Write                 |

## Access conditions
Let's address the elephant in the room: The access conditions. During my research, I found that many people struggled to make sense of the access condition section in the datasheet. Here is my attempt to explain it for easy to understand 🤞. 

You can use just 3 bit-combinations per block to control its permissions. In the official datasheet, this is represented using a notation like CX<sub>Y</sub> (C1₀, C1₂... C3₃) for the access bits. The first number (X) in this notation refers to the access bit number, which ranges from 1 to 3, each corresponding to a specific permission type. However, the meaning of these permissions varies depending on whether the block is a data block or a trailer block. The second number (Y) in the subscript denotes the relative block number, which ranges from 0 to 3.

### Table 1: Access conditions for the sector trailer
In the original datasheet, the subscript number is not specified in the table. I have added the subscript "3", as the sector trailer is located at Block 3.

<div class="alert-box alert-box-info">
    <span class="icon"><i class="fa fa-info"></i></span>
    <div class="alert-content">
        <b class="alert-title">Readable Key</b>
        <p>If you can read the key, it cannot be used as an authentication key. Therefore, in this table, whenever Key B is readable, it cannot serve as the authentication key. If you've noticed, yes, the Key A can never be read.</p>
    </div>
</div>

<table class="table-bordered">
  <thead>
    <tr >
      <th colspan="3" rowspan="2">Access Bits</th>
      <th colspan="6">Access Condition for</th>
      <th rowspan="3">Remark</th>
    </tr>
    <tr >
      <th colspan="2">Key A</th>
      <th colspan="2">Access Bits</th>
      <th colspan="2">Key B</th>
    </tr>
    <tr >
      <th>C1<sub>3</sub></th>
      <th>C2<sub>3</sub></th>
      <th>C3<sub>3</sub></th>
      <th>Read</th>
      <th>Write</th>
      <th>Read</th>
      <th>Write</th>
      <th>Read</th>
      <th>Write</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>never</td>
      <td>key A</td>
      <td>key A</td>
      <td>never</td>
      <td>key A</td>
      <td>key A</td>
      <td>Key B may be read</td>
    </tr>
    <tr>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>never</td>
      <td>never</td>
      <td>key A</td>
      <td>never</td>
      <td>key A</td>
      <td>never</td>
      <td>Key B may be read</td>
    </tr>
    <tr>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>never</td>
      <td>key B</td>
      <td>key A|B</td>
      <td>never</td>
      <td>never</td>
      <td>key B</td>
      <td></td>
    </tr>
    <tr>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>never</td>
      <td>never</td>
      <td>key A|B</td>
      <td>never</td>
      <td>never</td>
      <td>never</td>
      <td></td>
    </tr>
    <tr>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>never</td>
      <td>key A</td>
      <td>key A</td>
      <td>key A</td>
      <td>key A</td>
      <td>key A</td>
      <td>Key B may be read; Default configuration</td>
    </tr>
    <tr>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>never</td>
      <td>key B</td>
      <td>key A|B</td>
      <td>key B</td>
      <td>never</td>
      <td>key B</td>
      <td></td>
    </tr>
    <tr>
      <td>1</td>
      <td>0</td>
      <td>1</td>
      <td>never</td>
      <td>never</td>
      <td>key A|B</td>
      <td>key B</td>
      <td>never</td>
      <td>never</td>
      <td></td>
    </tr>
    <tr>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>never</td>
      <td>never</td>
      <td>key A|B</td>
      <td>never</td>
      <td>never</td>
      <td>never</td>
      <td></td>
    </tr>
  </tbody>
</table>


**How to make sense out of this table?** 

It is a simple table showing the correlation between bit combinations and permissions.
 
For example:
Let's say you select "1 0	0" (3rd row in the table), then you can't read KeyA, KeyB. However, you can modify the KeyA as well as KeyB value with KeyB. You can Read Access Bits with either KeyA or KeyB. But, you can never modify the Access Bits.

Now, where should these bits be stored? We will place them in the 6th, 7th, and 8th bytes at a specific location, which will be explained shortly.

### Table 2: Access conditions for data blocks
This applies to all data blocks. The original datasheet does not include the subscript "Y", I have added it for context. Here, "Y" represents the block number (ranging from 0 to 2).

The default config here indicates that both Key A and Key B can perform all operations. However, as seen in the previous table, Key B is readable (in default config), making it unusable for authentication. Therefore, only Key A can be used.

<table class="table-bordered">
  <thead>
    <tr >
      <th colspan="3">Access Bits</th>
      <th colspan="4">Access Condition for</th>
      <th rowspan="2">Application</th>
    </tr>
    <tr >
      <th>C1<sub>Y</sub></th>
      <th>C2<sub>Y</sub></th>
      <th>C3<sub>Y</sub></th>
      <th>Read</th>
      <th>Write</th>
      <th>Increment</th>
      <th>Decrement,Transfer/Restore</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>key A|B</td>
      <td>key A|B</td>
      <td>key A|B</td>
      <td>key A|B</td>
      <td>Default configuration</td>
    </tr>
    <tr>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>key A|B</td>
      <td>never</td>
      <td>never</td>
      <td>never</td>
      <td>read/write block</td>
    </tr>
    <tr>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>key A|B</td>
      <td>key B</td>
      <td>never</td>
      <td>never</td>
      <td>read/write block</td>
    </tr>
    <tr>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>key A|B</td>
      <td>key B</td>
      <td>key B</td>
      <td>key A|B</td>
      <td>value block</td>
    </tr>
    <tr>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>key A|B</td>
      <td>never</td>
      <td>never</td>
      <td>key A|B</td>
      <td>value block</td>
    </tr>
    <tr>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>key B</td>
      <td>key B</td>
      <td>never</td>
      <td>never</td>
      <td>read/write block</td>
    </tr>
    <tr>
      <td>1</td>
      <td>0</td>
      <td>1</td>
      <td>key B</td>
      <td>never</td>
      <td>never</td>
      <td>never</td>
      <td>read/write block</td>
    </tr>
    <tr>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>never</td>
      <td>never</td>
      <td>never</td>
      <td>never</td>
      <td>read/write block</td>
    </tr>
  </tbody>
</table>
Note: "If KeyB can be read in the Sector Trailer, it can't be used for authentication. As a result, if the reader uses KeyB to authenticate a block with access conditions that uses KeyB, the card will refuse any further memory access after authentication."

**How to make sense out of this table?** 

It's similar to the previous one; it shows the relationship between bit combinations and permissions.

For example:
If you select "0 1 0" (2nd row in the table) and use this permission for block 1, you can use either KeyA or KeyB to read block 1. However, no other operations can be performed on block 1.  

The notation for this is as follows: the block number is written as a subscript to the bit labels (e.g., C1<sub>1</sub>, C2<sub>1</sub>, C3<sub>1</sub>). Here, the subscript "1" represents block 1. For the selected combination "0 1 0", this means:  
- C1<sub>1</sub> = 0  
- C2<sub>1</sub> = 1  
- C3<sub>1</sub> = 0  

These bits will also be placed in the 6th, 7th, and 8th bytes at a specific location, which will be explained shortly.

### Table 3: Access conditions table
Let's colorize the original table to better visualize what each bit represents. The 7th and 3rd bits in each byte are related to the sector trailer. The 6th and 2nd bits correspond to Block 2. The 5th and 1st bits are associated with Block 1. The 4th and 0th bits are related to Block 0.

The overline on the notation indicates inverted values. This means that if the CX<sub>y</sub> value is 0, then <span style="text-decoration: overline;">CX</span><sub>y</sub> becomes 1.

<table >
  <thead>
    <tr>
      <th>Byte</th>
      <th>7</th>
      <th>6</th>
      <th>5</th>
      <th>4</th>
      <th>3</th>
      <th>2</th>
      <th>1</th>
      <th>0</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Byte 6</td>
      <td style="color:white;background-color:#8B0000"><span style="text-decoration: overline;">C2</span><sub>3</sub></td>
      <td style="color:white;background-color:#002D62"><span style="text-decoration: overline;">C2</span><sub>2</sub></td>
      <td style="color:white;background-color:#78184A" ><span style="text-decoration: overline;">C2</span><sub>1</sub></td>
      <td style="color:white;background-color:#234F1E"><span style="text-decoration: overline;">C2</span><sub>0</sub></td>
      <td style="color:white;background-color:#8B0000"><span style="text-decoration: overline;">C1</span><sub>3</sub></td>
      <td style="color:white;background-color:#002D62"><span style="text-decoration: overline;">C1</span><sub>2</sub></td>
      <td style="color:white;background-color:#78184A"><span style="text-decoration: overline;">C1</span><sub>1</sub></td>
      <td style="color:white;background-color:#234F1E"><span style="text-decoration: overline;">C1</span><sub>0</sub></td>
    </tr>
    <tr>
      <td>Byte 7</td>
      <td style="color:white;background-color:#8B0000">C1<sub>3</sub></td>
      <td style="color:white;background-color:#002D62">C1<sub>2</sub></td>
      <td style="color:white;background-color:#78184A">C1<sub>1</sub></td>
      <td style="color:white;background-color:#234F1E">C1<sub>0</sub></td>
      <td style="color:white;background-color:#8B0000"><span style="text-decoration: overline;">C3</span><sub>3</sub></td>
      <td style="color:white;background-color:#002D62"><span style="text-decoration: overline;">C3</span><sub>2</sub></td>
      <td style="color:white;background-color:#78184A"><span style="text-decoration: overline;">C3</span><sub>1</sub></td>
      <td style="color:white;background-color:#234F1E"><span style="text-decoration: overline;">C3</span><sub>0</sub></td>
    </tr>
    <tr>
      <td>Byte 8</td>
      <td style="color:white;background-color:#8B0000">C3<sub>3</sub></td>
      <td style="color:white;background-color:#002D62">C3<sub>2</sub></td>
      <td style="color:white;background-color:#78184A">C3<sub>1</sub></td>
      <td style="color:white;background-color:#234F1E">C3<sub>0</sub></td>
      <td style="color:white;background-color:#8B0000">C2<sub>3</sub></td>
      <td style="color:white;background-color:#002D62">C2<sub>2</sub></td>
      <td style="color:white;background-color:#78184A">C2<sub>1</sub></td>
      <td style="color:white;background-color:#234F1E">C2<sub>0</sub></td>
    </tr>
  </tbody>
</table>

The default access bit "FF 07 80". Let's try to understand what it means. 
<table border="1">
  <thead>
    <tr>
      <th>Byte</th>
      <th>7</th>
      <th>6</th>
      <th>5</th>
      <th>4</th>
      <th>3</th>
      <th>2</th>
      <th>1</th>
      <th>0</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Byte 6</td>
      <td style="color:white;background-color:#8B0000">1</td>
      <td style="color:white;background-color:#002D62">1</td>
      <td style="color:white;background-color:#78184A" >1</td>
      <td style="color:white;background-color:#234F1E">1</td>
      <td style="color:white;background-color:#8B0000">1</td>
      <td style="color:white;background-color:#002D62">1</td>
      <td style="color:white;background-color:#78184A">1</td>
      <td style="color:white;background-color:#234F1E">1</td>
    </tr>
    <tr>
      <td>Byte 7</td>
      <td style="color:white;background-color:#8B0000">0</td>
      <td style="color:white;background-color:#002D62">0</td>
      <td style="color:white;background-color:#78184A">0</td>
      <td style="color:white;background-color:#234F1E">0</td>
      <td style="color:white;background-color:#8B0000">0</td>
      <td style="color:white;background-color:#002D62">1</td>
      <td style="color:white;background-color:#78184A">1</td>
      <td style="color:white;background-color:#234F1E">1</td>
    </tr>
    <tr>
      <td>Byte 8</td>
      <td style="color:white;background-color:#8B0000">1</td>
      <td style="color:white;background-color:#002D62">0</td>
      <td style="color:white;background-color:#78184A">0</td>
      <td style="color:white;background-color:#234F1E">0</td>
      <td style="color:white;background-color:#8B0000">0</td>
      <td style="color:white;background-color:#002D62">0</td>
      <td style="color:white;background-color:#78184A">0</td>
      <td style="color:white;background-color:#234F1E">0</td>
    </tr>
  </tbody>
</table>

We can derive the CX<sub>Y</sub> values from the table above. Notice that only C3<sub>3</sub> is set to 1, while all other values are 0. Now, refer to Table 1 and Table 2 to understand which permission this corresponds to.
 
<table border="1">
  <thead>
    <tr>
      <th>Block</th>
      <th>C1<sub>Y</sub></th>
      <th>C2<sub>Y</sub></th>
      <th>C3<sub>Y</sub></th>
      <th>Access</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Block 0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>All permissions with Key A</td>
    </tr>
    <tr>
      <td>Block 1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>All permissions with Key A</td>
    </tr>
    <tr>
      <td>Block 2</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>All permissions with Key A</td>
    </tr>
    <tr>
      <td>Block 3 (Trailer)</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>You can write Key A using Key A. Access Bits and Key B can only be read and written using Key A. </td>
    </tr>
  </tbody>
</table>

Since Key B is readable, you cannot use it for authentication. 

### Calculator on next page
Still confused? Use the calculator on the next page to experiment with different combinations. Adjust the permissions for each block and observe how the Access Bits values change accordingly.
 
 ### Reference
  - [11th page of the datasheet](https://www.nxp.com/docs/en/data-sheet/MF1S50YYX_V1.pdf)
