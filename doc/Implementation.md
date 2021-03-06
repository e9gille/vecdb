# IMPLEMENTATION #

`vecdb`is an "inverted" database written in and for Dyalog APL, based on the ⎕MAP facility for mapping APL arrays to "flat" files. 

Data Types supported are

| Name | Description                                                |
|------|------------------------------------------------------------|
| B    | Boolean; 1 bit per item (0 or 1)                           |
| I1   | 1-Byte Integer (-128 to +127)                              |
| I2   | 2-Byte Integer (-32,768 to +32,767)                        |
| I4   | 4-Byte Integer (+/- 2,147,483,647/8)                       |
| F    | IEEE Double-Precision Floating Point (+/- ~1.797E308)      |
| C    | VarChar - up to 32,767 different strings indexed by an I2  |

A single-byte character type is planned - as C but indexed by an I1, allowing only 127 different strings. The current proposal is to denote this type "c".

### File Formats ###
Data is stored as APL vectors, each one mapping to a single file representing a numeric vector of uniform type, with the file extension *.vector*. In the case of the "C" type which contains variable length characters arrays, the serialised form (created using 220⌶) of a vector of character vectors is stored in a file with extension *.symbol*, and a 2-byte integer of indices into this is stored in the corresponding *.vector* file. 

#### Blocking ####
Since mapped arrays cannot grow dynamically, the files are over-allocated using a configurable *BlockSize* (*NumBlocks* tracks the number of blocks in use). All vectors have the same length; the number of records actually in used is tracked separately. When a new block is required, all maps are expunged, ⎕NAPPEND is used to add a block to each file, and the maps are re-created.

#### Meta Data ####
Meta-data is stored in a Dyalog Component File "meta.vecdb", which contains data which does not change during normal operation of the database:
 
| Cn# | Contents             | Example                                        |
|-----|----------------------|------------------------------------------------|
|   1 | Version number       | 'vecdb 0.2.0                                   |
|   2 | Description          | a char vec                                     |
|   3 | Unused               | Used to contain number of records              |
|   4 | Properties           | ('Name' 'BlockSize')('TestDB1' 10000)          |
|   5 | Col names & types    | ('Stock' 'Price') ((,'C') 'F')                |
|   6 | Shard folders        | 'c:\mydb\shard1\' 'c:\mydb\shard2\             |
|   7 | Shard fn and cols    | '{1+2|⎕UCS ⊃¨⍵}'  (,1)                        |

#### Sharding ####

`vecdb` allows the database to be *horizontally partitioned* into *shards*, based on the values of any selection of fields. If a database is created without sharding, data files are created in the same folder that the meta.vecdb file is in. Sharding is specified by passing suitable options to the vecdb constructor. The above example was set up using the following code. For a more advanced example, see the function `TestVecdb.Sharding`:

      columns←'Name' 'BlockSize' ⋄ types←,¨'C' 'F'
      data←('IBM' 'AAPL' 'MSFT' 'DYALOG')(160.97 112.6 47.21 999.99)
      options←⎕NS''
      options.BlockSize←10000
      options.ShardFolders←'c:\mydb\shard'∘,¨'12'
      options.(ShardFn ShardCols)←'{1+2|⎕UCS ⊃¨⍵}' 1 
      params←'TestDB1' 'c:\mydb' columns types options data
      mydb←⎕NEW vecdb params

In the above example, the database has two shards, based on whether the first character of the Stock name has an odd or even Unicode number.

Each shard is stored in a separate folder which contains the *.vector* files described above, plus a file "counters.vecdb" which currently contains a single 8-byte floating-point value which is the number of active records in the shard (the maximum number of records in a shard is limited to 2*48).

Note that the *.symbol* files are not sharded: The complete list of unique strings for a column is shared between the shards, and resides in main database folder.

The complete set of files which would be created by the above example would be along the lines of:

    Directory of c:\mydb\shardtest
    
    04/01/2015  21:35     122 1.symbol         // Symbols for Name column
    04/01/2015  21:35   2,576 meta.vecdb       // Meta data
    
     Directory of c:\mydb\db\shardtest\Shard1
    
    04/01/2015  21:35  20,000 1.vector         // 1 block of I2 symbol pointers
    04/01/2015  21:35  80,000 2.vector         // 1 block of Floating-point prices
    04/01/2015  21:35       8 counters.vecdb   // Used record counter (contains 3)
    
     Directory of c:\mydb\vecdb\shardtest\Shard2
    
    04/01/2015  21:35  20,000 1.vector         // As Shard1
    04/01/2015  21:35  80,000 2.vector         // As Shard1
    04/01/2015  21:35       8 counters.vecdb   // record counter (1)
    