[![Donate](https://img.shields.io/badge/Donate-PayPal-green.svg)](https://www.paypal.com/donate?hosted_button_id=69L3MWGCKVMA6)
# MFT_Record_Viewer

*Designed to view the details of few sets of $MFT FILE records, basically as a research project & for tool verification.
Try not to load more than a few hundred records at a time inorder for the app to work properly.*

### A few Observations

$MFT is pretty well documented, but some bits and pieces are missing. Like:
The FILE record 2 byte 'allocation status' flags at offset 0x16 (22) are 4:

  *  0x0001 = MFT_RECORD_IN_USE 
  *  0x0002 = MFT_RECORD_IS_DIRECTORY 
  *  0x0004 = MFT_RECORD_IN_EXTEND (i.e. in the $Extend directory) 
  *  0x0008 = MFT_RECORD_IS_VIEW_INDEX *(not set if file is index $I30)*
  
  And as noted in [here](https://opensource.apple.com/source/ntfs/ntfs-52/kext/ntfs_layout.h),
  ```C++
  /* 
   * The flag FILE_ATTR_DUP_FILENAME_INDEX_PRESENT is present in all 
   * FILENAME_ATTR attributes but not in the STANDARD_INFORMATION 
   * attribute of an mft record. 
   */ 
FILE_ATTR_DUP_FILE_NAME_INDEX_PRESENT  = cpu_to_le32(0x10000000), 
  /* Note, this is a copy of the corresponding bit from the mft record, 
     telling us whether this is a directory or not, i.e., whether it has 
     an index root attribute or not. */ 
  FILE_ATTR_DUP_VIEW_INDEX_PRESENT  = cpu_to_le32(0x20000000), 
  /* Note, this is a copy of the corresponding bit from the mft record, 
     telling us whether this file has a view index present (eg. object id 
     index, quota index, one of the security indexes or the encrypting 
     filesystem related indexes). */ 
```
     
meaning that when bit nr:<br> 
   1. is set to 1, the record is in-use.
   2. is set to 1, the record has an  $Index_Root attribute, thus making it a Directory. When this bit is set, the related $Filename Attribute flag is set to 'Directory' *(see below)*
   3. is set to 1, the FILE's location is in the $Extend directory
   4. is set to 1, the record has a View_Index *(eg. object id, index, quota index, one of the security indexes or the encrypting filesystem related indexes)*, excluding $I30. When this bit is set, the related $Filename Attribute flag is set to 'Index_view' *(see below)*
     
Further down the FILE record we get to the attributes, which are:

  * "0x00000000 - Unused"
  * "0x10000000 - $Standard_Information"
  * "0x20000000 - $Attribute_List"
  * "0x30000000 - $File_Name"
  * "0x40000000 - $Object_ID"
  * "0x50000000 - $Security_Descriptor"
  * "0x60000000 - $Volume_Name"
  * "0x70000000 - $Volume_Information"
  * "0x80000000 - $Data"
  * "0x90000000 - $Index_Root"
  * "0xA0000000 - $Index_Allocation"
  * "0xB0000000 - $Bitmap"
  * "0xC0000000 - $Reparse_Point"
  * "0xD0000000 - $EA_Information"
  * "0xE0000000 - $EA"
  * "0x00010000 - $Logged_Utility_Stream"

The common Attribute header has a one byte flag at offset 0x16 (22) from the start of each attribute. This is the 'Indexed' flag, and is referenced [here](https://opensource.apple.com/source/ntfs/ntfs-52/kext/ntfs_layout.h) *(line 696)*:

  ```C++
   {
    RESIDENT_ATTR_IS_INDEXED = 0x01, /* Attribute is referenced in an index
                (has implications for deleting and
                modifying the attribute). */
                }
```
which  I've seen ‘set to 1’ only in $File_Name attributes, so I presume it is linked to $I30.

According to [Microsoft](https://docs.microsoft.com/en-us/windows/win32/fileio/file-attribute-constants) the FILE record Attribute Flags *(constants)* are:

  Hex|Binary|Description
  ----------|---------------------------------------|------------------
  0x00000001|0000-0000-0000-0000-0000-0000-0000-0001|ReadOnly
  0x00000002|0000-0000-0000-0000-0000-0000-0000-0010|Hidden
  0x00000004|0000-0000-0000-0000-0000-0000-0000-0100|System
  0x00000010|0000-0000-0000-0000-0000-0000-0001-0000|Directory
  0x00000020|0000-0000-0000-0000-0000-0000-0010-0000|Archive
  0x00000040|0000-0000-0000-0000-0000-0000-0100-0000|Device
  0x00000080|0000-0000-0000-0000-0000-0000-1000-0000|Normal
  0x00000100|0000-0000-0000-0000-0000-0001-0000-0000|Temporary
  0x00000200|0000-0000-0000-0000-0000-0010-0000-0000|Sparse_File
  0x00000400|0000-0000-0000-0000-0000-0100-0000-0000|Reparse_Point
  0x00000800|0000-0000-0000-0000-0000-1000-0000-0000|Compressed
  0x00001000|0000-0000-0000-0000-0001-0000-0000-0000|Offline
  0x00002000|0000-0000-0000-0000-0010-0000-0000-0000|Not_Content_Indexed
  0x00004000|0000-0000-0000-0000-0100-0000-0000-0000|Encrypted
  0x00008000|0000-0000-0000-0000-1000-0000-0000-0000|Integrity Stream
  0x00010000|0000-0000-0000-0001-0000-0000-0000-0000|Virtual
  0x00020000|0000-0000-0000-0010-0000-0000-0000-0000|No_Scrub_Data
  0x00040000|0000-0000-0000-0100-0000-0000-0000-0000|Recall_On_Open
  0x00400000|0000-0000-0100-0000-0000-0000-0000-0000|Recall_On_DataAccess

The $File_Name Attribute flags are the same as in $Standard_definition *(above list)*, but with a couple of exceptionns. As noted above, there are two extra flags which 'copy' the respective flags of the record header. When one is set in the header, the same is set in the $File_Name attribute. These flags are:

  Hex|Binary|Description
  ----------|---------------------------------------|------------------
  0x10000000|0001-0000-0000-0000-0000-0000-0000-0000|Directory
  0x20000000|0010-0000-0000-0000-0000-0000-0000-0000|Index_view

and I pressume that the 0x00000010 'Directory' bit is not beeing used in the $File_Name attribute. 

  **[Reparse points and extended attributes are mutually exclusive](https://docs.microsoft.com/en-us/windows/win32/fileio/reparse-points?redirectedfrom=MSDN)**
  - When the $File_Name attribute flag “Reparse Point” (0x00000400) is set to 1, offset 0x3C (60) from the start of the $File_Name attribute shows the [Tag](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-fscc/c8e77b37-3909-4fe6-a4ea-2b9d423b1ee4) value (type of Reparse point) of the $Reparse_Point attribute in the record *(value should match 0ffset 0x00 for 4 bytes from the start of the $Reparse_Point attribute resident content)*.

     *("[Reparse point tag](https://docs.microsoft.com/en-us/windows/win32/fileio/reparse-points):  A unique identifier for a file system filter driver stored within a file's optional reparse point data that indicates the file system filter driver that performs additional filter-defined processing on a file during I/O operations.")*

  - When the $File_Name attribute flag "Reparse Point” (0x400) is set to 0, flag "Recall_On_Open" (0x00040000) is set to 1, and there is no '$Reparse_point' attribute in the record, but there is an $EA present, offset 0x3C (60) from the start of the $File_Name attribute shows the 32bit size of the buffer needed for the $EA attribute.

._
