## elfutils
### libelf
#### elf.h
[ELF header Man page](https://man7.org/linux/man-pages/man5/elf.5.html)

##### ELF header (Ehdr)
> EI_NIDENT: This array of bytes specifies how to interpret the file, independent of the processor or the file's remaining contents. Within this array everything is named by macros, which start with the prefix EI_ and may contain values which start with the prefix ELF.
```
#define EI_NIDENT (16)

typedef struct
{
  unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
  Elf32_Half	e_type;			/* Object file type */
  Elf32_Half	e_machine;		/* Architecture */
  Elf32_Word	e_version;		/* Object file version */
  Elf32_Addr	e_entry;		/* Entry point virtual address */
  Elf32_Off	e_phoff;		/* Program header table file offset */
  Elf32_Off	e_shoff;		/* Section header table file offset */
  Elf32_Word	e_flags;		/* Processor-specific flags */
  Elf32_Half	e_ehsize;		/* ELF header size in bytes */
  Elf32_Half	e_phentsize;		/* Program header table entry size */
  Elf32_Half	e_phnum;		/* Program header table entry count */
  Elf32_Half	e_shentsize;		/* Section header table entry size */
  Elf32_Half	e_shnum;		/* Section header table entry count */
  Elf32_Half	e_shstrndx;		/* Section header string table index */
} Elf32_Ehdr;

typedef struct
{
  unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
  Elf64_Half	e_type;			/* Object file type */
  Elf64_Half	e_machine;		/* Architecture */
  Elf64_Word	e_version;		/* Object file version */
  Elf64_Addr	e_entry;		/* Entry point virtual address */
  Elf64_Off	e_phoff;		/* Program header table file offset */
  Elf64_Off	e_shoff;		/* Section header table file offset */
  Elf64_Word	e_flags;		/* Processor-specific flags */
  Elf64_Half	e_ehsize;		/* ELF header size in bytes */
  Elf64_Half	e_phentsize;		/* Program header table entry size */
  Elf64_Half	e_phnum;		/* Program header table entry count */
  Elf64_Half	e_shentsize;		/* Section header table entry size */
  Elf64_Half	e_shnum;		/* Section header table entry count */
  Elf64_Half	e_shstrndx;		/* Section header string table index */
} Elf64_Ehdr;

```

##### Section header (Shdr)

```
typedef struct
{
  Elf32_Word	sh_name;		/* Section name (string tbl index) */
  Elf32_Word	sh_type;		/* Section type */
  Elf32_Word	sh_flags;		/* Section flags */
  Elf32_Addr	sh_addr;		/* Section virtual addr at execution */
  Elf32_Off	sh_offset;		/* Section file offset */
  Elf32_Word	sh_size;		/* Section size in bytes */
  Elf32_Word	sh_link;		/* Link to another section */
  Elf32_Word	sh_info;		/* Additional section information */
  Elf32_Word	sh_addralign;		/* Section alignment */
  Elf32_Word	sh_entsize;		/* Entry size if section holds table */
} Elf32_Shdr;

typedef struct
{
  Elf64_Word	sh_name;		/* Section name (string tbl index) */
  Elf64_Word	sh_type;		/* Section type */
  Elf64_Xword	sh_flags;		/* Section flags */
  Elf64_Addr	sh_addr;		/* Section virtual addr at execution */
  Elf64_Off	sh_offset;		/* Section file offset */
  Elf64_Xword	sh_size;		/* Section size in bytes */
  Elf64_Word	sh_link;		/* Link to another section */
  Elf64_Word	sh_info;		/* Additional section information */
  Elf64_Xword	sh_addralign;		/* Section alignment */
  Elf64_Xword	sh_entsize;		/* Entry size if section holds table */
} Elf64_Shdr;

```

##### Program header (Phdr)
```
typedef struct
{
  Elf32_Word	p_type;			/* Segment type */
  Elf32_Off	p_offset;		/* Segment file offset */
  Elf32_Addr	p_vaddr;		/* Segment virtual address */
  Elf32_Addr	p_paddr;		/* Segment physical address */
  Elf32_Word	p_filesz;		/* Segment size in file */
  Elf32_Word	p_memsz;		/* Segment size in memory */
  Elf32_Word	p_flags;		/* Segment flags */
  Elf32_Word	p_align;		/* Segment alignment */
} Elf32_Phdr;

typedef struct
{
  Elf64_Word	p_type;			/* Segment type */
  Elf64_Word	p_flags;		/* Segment flags */
  Elf64_Off	p_offset;		/* Segment file offset */
  Elf64_Addr	p_vaddr;		/* Segment virtual address */
  Elf64_Addr	p_paddr;		/* Segment physical address */
  Elf64_Xword	p_filesz;		/* Segment size in file */
  Elf64_Xword	p_memsz;		/* Segment size in memory */
  Elf64_Xword	p_align;		/* Segment alignment */
} Elf64_Phdr;

typedef struct
{
  Elf64_Word	st_name;		/* Symbol name (string tbl index) */
  unsigned char	st_info;		/* Symbol type and binding */
  unsigned char st_other;		/* Symbol visibility */
  Elf64_Section	st_shndx;		/* Section index */
  Elf64_Addr	st_value;		/* Symbol value */
  Elf64_Xword	st_size;		/* Symbol size */
} Elf64_Sym;

typedef struct
{
  Elf64_Addr	r_offset;		/* Address */
  Elf64_Xword	r_info;			/* Relocation type and symbol index */
} Elf64_Rel;

typedef struct
{
  Elf64_Sxword	d_tag;			/* Dynamic entry type */
  union
    {
      Elf64_Xword d_val;		/* Integer value */
      Elf64_Addr d_ptr;			/* Address value */
    } d_un;
} Elf64_Dyn;

typedef struct
{
  Elf64_Word l_name;		/* Name (string table index) */
  Elf64_Word l_time_stamp;	/* Timestamp */
  Elf64_Word l_checksum;	/* Checksum */
  Elf64_Word l_version;		/* Interface version */
  Elf64_Word l_flags;		/* Flags */
} Elf64_Lib;


```

#### libelf internal data structures
```
libelf/libelfP.h

struct Elf_Scn
{
  Elf_Data_List data_list;	/* List of data buffers.  */
  Elf_Data_List *data_list_rear; /* Pointer to the rear of the data list. */
  Elf_Data_Scn rawdata;		/* Uninterpreted data of the section.  */
  int data_read;		/* Nonzero if the section was created by the user or if the data from the file/memory is read.  */
  int shndx_index;		/* Index of the extended section index table for this symbol table  */
  size_t index;			/* Index of this section.  */
  struct Elf *elf;		/* The underlying ELF file.  */
  union
  {
    Elf32_Shdr *e32;		/* Pointer to 32bit section header.  */
    Elf64_Shdr *e64;		/* Pointer to 64bit section header.  */
  } shdr;
  unsigned int shdr_flags;	/* Section header modified?  */
  unsigned int flags;		
  char *rawdata_base;		/* The unmodified data of the section.  */
  char *data_base;		/* The converted data of the section.  */
  char *zdata_base;		/* The uncompressed data of the section.  */
  size_t zdata_size;		/* If zdata_base != NULL, the size of data.  */
  size_t zdata_align;		/* If zdata_base != NULL, the addralign.  */
  struct Elf_ScnList *list;	/* Pointer to the section list element the  data is in.  */
};

typedef struct Elf_ScnList
{
  unsigned int cnt;		/* Number of elements of 'data' used.  */
  unsigned int max;		/* Number of elements of 'data' allocated.  */
  struct Elf_ScnList *next;	/* Next block of sections.  */
  struct Elf_Scn data[0];	/* Section data.  */
} Elf_ScnList;


typedef struct Elf_Data_List
{
  Elf_Data_Scn data;
  struct Elf_Data_List *next;
  int flags;
} Elf_Data_List;


typedef struct
{
  Elf_Data d;
  Elf_Scn *s;
} Elf_Data_Scn;


/* The ELF descriptor.  */
struct Elf
{
  void *map_address;
  Elf *parent;  /* When created for an archive member this points to the descriptor for the archive. */
  Elf *next;    /* Used in list of archive descriptors.  */
  Elf_Kind kind;  /* What kind of file is underneath (ELF file, archive...).  */
  Elf_Cmd cmd;    /* Command used to create this descriptor.  */
  unsigned int class;   /* The binary class.  */
  int fildes;
  int64_t start_offset;
  size_t maximum_size;
  int flags;
  int ref_count;
  rwlock_define (,lock);
  union
  {
    struct
    {
      void *ehdr;
      void *shdr;
      void *phdr;

      Elf_ScnList *scns_last;	/* Last element in the section list. If NULL the data has not yet been read from the file.  */
      Elf_Data_Chunk *rawchunks; /* List of elf_getdata_rawchunk results.  */
      unsigned int scnincr;	/* Number of sections allocate the last time.  */
      int ehdr_flags;		/* Flags (dirty) for ELF header.  */
      int phdr_flags;		/* Flags (dirty|malloc) for program header.  */
      int shdr_malloced;	/* Nonzero if shdr array was allocated.  */
      off_t sizestr_offset;	/* Offset of the size string in the parent
				   if this is an archive member.  */
    } elf;

    struct
    {
      Elf32_Ehdr *ehdr;
      Elf32_Shdr *shdr;		/* Used when reading from a file.  */
      Elf32_Phdr *phdr;		/* Pointer to the program header array.  */
      Elf_ScnList *scns_last;
      Elf_Data_Chunk *rawchunks; /* List of elf_getdata_rawchunk results.  */
      unsigned int scnincr;
      int ehdr_flags;
      int phdr_flags;
      int shdr_malloced;
      int64_t sizestr_offset;
      Elf32_Ehdr ehdr_mem;
      char __e32scnspad[sizeof (Elf64_Ehdr) - sizeof (Elf32_Ehdr)];
      /* The section array.  */
      Elf_ScnList scns;
    } elf32;

    struct
    {
      Elf64_Ehdr *ehdr;
      Elf64_Shdr *shdr;
      Elf64_Phdr *phdr;
      Elf_ScnList *scns_last;
      Elf_Data_Chunk *rawchunks;
      unsigned int scnincr;
      int ehdr_flags;
      int phdr_flags;
      int shdr_malloced;
      Elf64_Ehdr ehdr_mem;
      /* The section array.  */
      Elf_ScnList scns;
    } elf64;

    struct
    {
      Elf *children;		/* List of all descriptors for this archive. */
      Elf_Arsym *ar_sym;	/* Symbol table returned by elf_getarsym.  */
      size_t ar_sym_num;	/* Number of entries in `ar_sym'.  */
      char *long_names;		/* If no index is available but long names are used this elements points to the data.*/
      size_t long_names_len;	/* Length of the long name table.  */
      int64_t offset;		/* Offset in file we are currently at. elf_next() advances this to the next member of the archive.  */
      Elf_Arhdr elf_ar_hdr;	/* Structure returned by 'elf_getarhdr'.  */
      struct ar_hdr ar_hdr;	/* Header read from file.  */
      char ar_name[16];		/* NUL terminated ar_name of elf_ar_hdr.  */
      char raw_name[17];	/* This is a buffer for the NUL terminated named raw_name used in the elf_ar_hdr.  */
    } ar;
  } state;
};
```

#### libelf.h
```
typedef struct
{
  void *d_buf;			/* Pointer to the actual data.  */
  Elf_Type d_type;		/* Type of this piece of data.  */
  unsigned int d_version;	/* ELF version.  */
  size_t d_size;		/* Size in bytes.  */
  int64_t d_off;		/* Offset into section.  */
  size_t d_align;		/* Alignment in section.  */
} Elf_Data;
```

#### elf_begin.c
```
elf_begin
  case ELF_C_READ_MMAP_PRIVATE, ELF_C_READ, ELF_C_READ_MMAP
    lock_dup_elf
    dup_elf if with ref
      read_file
        if use_map && parent == null
          mmap
          __libelf_read_mmaped_file
            ELF_K_ELF
              file_read_elf
                get_shnum // get the shnum from Ehdr
                allocate_elf
                populate struct Elf
            ELF_K_AR
              file_read_ar
        else
          read_unmmaped_file    
            pread_retry ==> pread reads Ehdr
            file_read_elf
  case ELF_C_WRITE, ELF_C_WRITE_MMAP
    write_file
      allocate_elf

```

#### elf_getscn.c
```
elf->state.elf32/64.scns
// somehow get the Elf_Scn from Elf_ScnList, need to refer ELF Documentation and struct Elf

```

#### elf_getdata.c
```
elf_getdata ==> __elf_getdata_rdlock
  scn->data_list.data.d or scn->data_list.next.data.d

```

#### elf_strptr.c
```
  Elf_Scn.rawdata_base[offset]
```

#### gelf_getsym.c
```
  ((GElf_Sym *) data->d_buf)[ndx]
```
