# MIFARE
MIFARE is a series of integrated circuit (IC) chips used in contactless smart cards and proximity cards, developed by NXP Semiconductors. MIFARE cards follow ISO/IEC 14443A standards and use encryption methods such as Crypto-1 algorithm. The most common family is MIFARE Classic, with a subtype called [MIFARE Classic EV1](https://www.nxp.com/products/rfid-nfc/mifare-hf/mifare-classic/mifare-classic-ev1-1k-4k:MF1S50YYX_V1).

## Memory Layout
The MIFARE Classic 1K card is divided into 16 sectors, with each sector containing 4 blocks. Each block can hold up to 16 bytes, resulting in a total memory capacity of 1KB.

16 sectors × 4 blocks/sector × 16 bytes/block = 1024 bytes = 1KB

<a href="./images/mifare-memory-layout.jpg"><img style="display: block; margin: auto;" alt="MIFARE Memory layout" src="./images/mifare-memory-layout.jpg"/></a>

### Sector Trailer

The last block of each sector, known as the "trailer" holds two secret keys and programmable access conditions for the blocks within that sector. Each sector has its own pair of keys (KeyA and KeyB), enabling support for multiple applications with a key hierarchy.

<div class="alert-box alert-box-info">
    <span class="icon"><i class="fa fa-info"></i></span>
    <div class="alert-content">
        <b class="alert-title">Default Keys</b>
        <p>The MIFARE Classic 1K card is pre-configured with the default key FF FF FF FF FF FF for both KeyA and KeyB.  When reading the trailer block, KeyA values are returned as all zeros (00 00 00 00 00 00), while KeyB returned as it is.</p>
    </div>
</div>


 By default, the access bytes (6, 7, and 8 of the trailer) are set to FF 07 80h. You can refer the 10th page for the [datasheet](https://www.nxp.com/docs/en/data-sheet/MF1S50YYX_V1.pdf) for more information. And the 9th byte can be used for storing data.

<table border="1" cellspacing="0" cellpadding="5" style="border-collapse: collapse; width: 100%; text-align: center;">
  <thead>
    <tr>
      <th rowspan="2" style="background-color: #607D8B; color: #000;">Byte Number</th>
    </tr>
    <tr>
      <th style="background-color: #1B5E20;">0</th>
      <th style="background-color: #1B5E20;">1</th>
      <th style="background-color: #1B5E20;">2</th>
      <th style="background-color: #1B5E20;">3</th>
      <th style="background-color: #1B5E20;">4</th>
      <th style="background-color: #1B5E20;">5</th>
      <th style="background-color: #FF6F00;">6</th>
      <th style="background-color: #FF6F00;">7</th>
      <th style="background-color: #FF6F00;">8</th>
      <th style="background-color: #FF6F00;">9</th>
      <th style="background-color: #2962FF;">10</th>
      <th style="background-color: #2962FF;">11</th>
      <th style="background-color: #2962FF;">12</th>
      <th style="background-color: #2962FF;">13</th>
      <th style="background-color: #2962FF;">14</th>
      <th style="background-color: #2962FF;">15</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="background-color: #607D8B; color: #000;">Description</td>
      <td colspan="6" style="background-color: #1B5E20; color: #000;">KEY A</td>
      <td colspan="3" style="background-color: #FF6F00; color: #000;">Access Bits</td>
      <td style="background-color: #FF6F00; color: #000;">USER Data</td>
      <td colspan="6" style="background-color: #2962FF; color: #000;">KEY B</td>
    </tr>
    <tr>
      <td style="background-color: #607D8B; color: #000;">Default Data</td>
      <td style="background-color: #1B5E20;">FF</td>
      <td style="background-color: #1B5E20;">FF</td>
      <td style="background-color: #1B5E20;">FF</td>
      <td style="background-color: #1B5E20;">FF</td>
      <td style="background-color: #1B5E20;">FF</td>
      <td style="background-color: #1B5E20;">FF</td>
      <td style="background-color: #FF6F00;">FF</td>
      <td style="background-color: #FF6F00;">07</td>
      <td style="background-color: #FF6F00;">80</td>
      <td style="background-color: #FF6F00;">69</td>
      <td style="background-color: #2962FF;">FF</td>
      <td style="background-color: #2962FF;">FF</td>
      <td style="background-color: #2962FF;">FF</td>
      <td style="background-color: #2962FF;">FF</td>
      <td style="background-color: #2962FF;">FF</td>
      <td style="background-color: #2962FF;">FF</td>
    </tr>
  </tbody>
</table>



### Manufacturer Block

The first block (block 0) of the first sector(sector 0) contains IC manufacturer's data including the UID. This block is write-protected.  

### Data Block

Each sector has a trailer block, so only 3 blocks can be used for data storage in each sector. However, the first sector only has 2 usable blocks because the first block stores manufacturer data.

To read or write the data, you first need to authenticate with either Key A or Key B of that sector. 

The data blocks can be further classified into two categories based on the access bits(we will explain about it later). 
- read/write block: These are standard data blocks that allow basic operations such as reading and writing data.
- value block: These blocks are ideal for applications like electronic purses, where they are commonly used to store numeric values, such as account balances. So, you can perform incrementing (e.g., adding $10 to a balance) or decrementing (e.g., deducting $5 for a transaction). 

## Reference
- Datasheet: [MIFARE Classic EV1 1K - Mainstream contactless smart card IC for fast and easy solution development](https://www.nxp.com/docs/en/data-sheet/MF1S50YYX_V1.pdf)
