## JOS文件系统

- 大多数UNIX文件系统将磁盘分为*inode*区和*data*区。其中每个文件一个*inode*，文件的*inode*包含文件的关键*meta-data*，比如文件的状态信息和指向文件数据块的指针等等。文件目录项(Directory entries)包含文件名和该文件的*inode*。

- 如果多个文件目录项(Directory entries)指向一个*inode*，那么我们就说该文件为**硬连接(*hard-linked*)**，**$\textcolor{red}{之后了解一下这些概念}$**

- JOS不支持硬连接，所以不需要*inode*，所以JOS中文件目录项中也包含*inode*的内容

- 在一般系统中，用户向进行文件操作，必须进行系统调用。JOS中将这些文件操作交给专门的*file system*(简称FS)这样一个进程来实现，如下图所示，普通进行需要对文件进行操作时，需要和FS进程进行IPC，普通进程给FS进程发送操作文件所必要的信息（比如文件的打开文件号、读写文件的大小等等），然后FS进程进行文件操作后以IPC的方式传给普通进程

  <img src="https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202309061906023.png" alt="1693998406895" style="zoom: 80%;" />

### JOS的文件系统结构

- 在JOS文件系统中，文件目录项保存着该文件的数据块等信息，这些信息叫做*meta-data*

- 下图为JOS的文件系统结构图，可知JOS用`File`这个数据结构来表示一个文件的*meta-data*，包括文件名、文件大小、文件数据块号。其中*Direct block*直接表示该文件的某个数据块，而*Indirect block*则是指向另一个磁盘块，该块的内容都是磁盘号，这些磁盘号指向了该文件的数据块。

  <img src="https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202309061915243.png" alt="1693998900349" style="zoom:80%;" />

- 一个`File`的*Direct block*为10个，一个磁盘块为4KB，磁盘号占4B，所以在**JOS中一个文件最大为(10 + 1024) * 4KB，大概4MB。**注意这些*Direct block*和*Indirect block*都是磁盘块。

- JOS中目录和普通文件的*meta-data*都是用`File`这个数据结构来表示的，其中**目录的`File`所指向的那些*File data blocks*不再是文件数据，而是一个个`File`型的数据**，因为一个目录里存的是这个目录地下的文件或目录

### JOS磁盘布局

- 在LAB 1中，我们给*kernel.img*分配了10000个页作为磁盘，在这一个LAB中，我们又创建了一个磁盘，位于*obj/fs/fs.img*中，我们在其MakeFile文件中可以看到，该文件是通过执行fsformat这个文件来生成的。

  ```
  $(OBJDIR)/fs/clean-fs.img: $(OBJDIR)/fs/fsformat $(FSIMGFILES)
     @echo + mk $(OBJDIR)/fs/clean-fs.img
     $(V)mkdir -p $(@D)
     $(V)$(OBJDIR)/fs/fsformat $(OBJDIR)/fs/clean-fs.img 1024 $(FSIMGFILES)
  ```

- 该函数创建了大小为1024 * 4KB = 4MB大小的*obj/fs/fs.img*，并且按照下图所示的布局来初始化磁盘。其中第0块不用，第一块放*Superbolck*，第二块放*bitmap*，最后在磁盘上放了一些文件，并将这些文件的*meta-data*写入根目录'/'中。

  <img src="https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202309061937113.png" alt="1694000095309" style="zoom: 67%;" />

- 在磁盘上放文件的过程其实就是将一个文件的内容写入*obj/fs/fs.img*的过程。我们需要先c语言库函数mmap来将*obj/fs/fs.img*的磁盘地址空间映射到某一段进程虚拟地址空间中，这样我们先往虚拟地址空间写内容，然后使用库函数msync来将映射区修改的页刷到磁盘中。
- 然后给写入磁盘的每个文件分配一个`File`来管理*meta-data*，最后将这些`File`数据挂到根目录的`File`下，即根目录的`File`指向根目录下文件的`File`所在的磁盘块。

### Block Cache

- 我们在内存中将0x10000000 (`DISKMAP`) 到 0xD0000000 (`DISKMAP+DISKMAX`)这一块3GB的虚拟内存空间称为*block chche*，这块3GB的空间与磁盘空间相对应，比如磁盘块0对应的虚拟内存地址为0x10000000，磁盘块1为0x10001000

- 这样的好处是，比如当我们想要写入磁盘时，直接将该虚拟地址转为磁盘块号，然后写入磁盘。当我们读磁盘时，将磁盘块号转为虚拟地址，读到对应的地址处。当我们想要访问一个文件时，其`File`中记录的磁盘块转为虚拟地址，然后直接用该虚拟地址访问。方便磁盘的读写管理。

- 也就是JOS实现了 **虚拟地址-物理地址的映射** 与 **虚拟地址-磁盘块号的映射**

- **$\textcolor{red}{注意：}$**虚拟地址-磁盘块号这个映射关系只有FS进程有，其他进程没有这个映射关系

- 当我们访问一个文件时，有时候该文件的内容所在的磁盘块并不在内存，也就是没有给虚拟地址分配物理地址，那么就会触发页错误，我们实现了FS里的页错误处理程序fs/bc.c里的bc_pgfaul，我们先给该虚拟地址分配物理内存，然后把该虚拟地址对应的磁盘内容写入内存。

  ```
  // Fault any disk block that is read in to memory by
  // loading it from disk.
  static void
  bc_pgfault(struct UTrapframe *utf)
  {
     void *addr = (void *) utf->utf_fault_va;
      // 首先磁盘号和虚拟地址有一一对应的关系
     uint32_t blockno = ((uint32_t)addr - DISKMAP) / BLKSIZE;
     int r;
  
     // Check that the fault was within the block cache region
     if (addr < (void*)DISKMAP || addr >= (void*)(DISKMAP + DISKSIZE))
        panic("page fault in FS: eip %08x, va %08x, err %04x",
              utf->utf_eip, addr, utf->utf_err);
  
     // Sanity check the block number.
     if (super && blockno >= super->s_nblocks)
        panic("reading non-existent block %08x\n", blockno);
  
     // Allocate a page in the disk map region, read the contents
     // of the block from the disk into that page.
     // Hint: first round addr to page boundary. fs/ide.c has code to read
     // the disk.
     //
     // LAB 5: you code here:
      addr = ROUNDDOWN(addr,PGSIZE);
      sys_page_alloc(0,addr,PTE_U | PTE_W | PTE_P);
      // 从blockno所在的扇区开始，读取BLKSECTS（8）个扇区
      if ((r = ide_read(blockno * BLKSECTS,addr,BLKSECTS)) < 0) {
          panic("ide_read error : %e",r);
      }
  
     // Clear the dirty bit for the disk block page since we just read the
     // block from disk
     if ((r = sys_page_map(0, addr, 0, addr, uvpt[PGNUM(addr)] & PTE_SYSCALL)) < 0)
        panic("in bc_pgfault, sys_page_map: %e", r);
  
     // Check that the block we read was allocated. (exercise for
     // the reader: why do we do this *after* reading the block
     // in?)
     if (bitmap && block_is_free(blockno))
        panic("reading free block %08x\n", blockno);
  }
  ```

- **$\textcolor{red}{注意：}$**JOS**还未实现*block cache*的驱逐机制**，也就是一旦发生页错误写入内存，该磁盘块就一直保留在内存中，即不释放物理内存，这在JOS中是可以运行的，因为每个虚拟地址惟一对应一个磁盘块，在内存足够的情况下可以一直占着物理内存。当然如果我们能够正确地调用`file_flush`来将*block cache*中的值刷新到磁盘中的话，直接访问*block cache*就是直接访问磁盘。

### File Operations

- fs/fs.c中实现了基本的文件操作，当然这些函数只能由FS进程来调用

#### 文件操作基本函数

- 查看第blockno个block是否为free。检查的是bitmap。如果一个block为free状态，说明该块没有分配给任何文件

  ```
  bool
  block_is_free(uint32_t blockno)
  ```

- 将第blockno个block的状态设为free，也就是修改bitmap，使得对应位为1

  ```
  void
  free_block(uint32_t blockno)
  ```

- 查找一个free的块，然后返回该块的磁盘号。其实就是遍历bitmap，找到为1的磁盘块

  ```
  // Search the bitmap for a free block and allocate it.  When you
  // allocate a block, immediately flush the changed bitmap block
  // to disk.
  int
  alloc_block(void)
  ```

- 初始化文件系统。将磁盘编号设置为1（也就是使用fs.img），然后设置文件系统的页错误处理程序，最后设置`super`和`bitmap`指针指向对应磁盘块的虚拟地址。

  ```
  void
  fs_init(void)
  ```

- 找到文件f第filebno的**块号**，然后赋给\*ppdiskbno。**$\textcolor{red}{注意：}$**如果文件f没有第filebno块，那么*ppdiskbno为0

  ```
  // 找到文件f第filebno的块号，然后赋给*ppdiskbno
  // 如果alloc，而且文件f此时没有f_indirect（也就是此时文件f的块数小于NDIRECT），那么给文件f分配一个indirect块
  // 注意：如果文件f没有第filebno块，那么*ppdiskbno为0
  static int
  file_block_walk(struct File *f, uint32_t filebno, uint32_t **ppdiskbno, bool alloc)
  ```

- 返回文件f第filebno块的**虚拟地址**。**$\textcolor{red}{注意：}$**如果文件f没有第filebno块，需要给其分配一个磁盘块，并且修改文件f所储存的磁盘块号

  ```
  // 返回文件f第filebno块的虚拟地址
  // 注意：这里如果*pdiskbno为0，那么说明文件f还没有第filebno块，需要给其分配一个磁盘块，并且修改文件f所储存的磁盘块号
  int
  file_get_block(struct File *f, uint32_t filebno, char **blk)
  ```

- 在目录dir中寻找文件名为name的文件。也就是遍历目录dir所指向的磁盘块，一个一个进行对比

  ```
  // Try to find a file named "name" in dir.  If so, set *file to it.
  static int
  dir_lookup(struct File *dir, const char *name, struct File **file)
  ```

- 从目录dir中找到一个空闲的`File`。先在已经分配好的磁盘块内找，看是否有空闲的位置。注意：目录所储存的磁盘块内存的都是File结构的数据；如果满了，那么给dir扩容，返回新磁盘的地址。**扩容方式**为，修改目录大小，并且查找该目录不存在的磁盘块的虚拟地址，这样会给该目录分配一个磁盘块

  ```
  // 从目录dir中找到一个空闲的File
  static int
  dir_alloc_file(struct File *dir, struct File **file)
  ```

- 查找依照路径path一个文件。path只能是绝对路径。从根目录'/'开始一级一级找

  ```
  // path只能是绝对路径
  // *pdir为目标文件或目录所在的目录
  // 若找到，那么*pf为目录文件或目录
  static int
  walk_path(const char *path, struct File **pdir, struct File **pf, char *lastelem)
  ```

#### 文件操作函数

- 创建文件，比如'/lzc/lzclzc/1.txt'，那么先寻找'lzc/lzclzc'这个路径是否存在，如果存在才进行创建

  通过`walk_path`函数找到所要创建的文件所在的目录，然后在该目录下调用`dir_alloc_file`来得到空闲的`File`，将该`File`返回。由于对目录文件进行了修改，所以最后调用`file_flush`将该目录文件刷到磁盘

  ```
  int
  file_create(const char *path, struct File **pf)
  ```

- 打开文件，得到该文件的`File`

  ```
  // 打开文件，如果文件存在，那么返回该文件的信息块到*pf中
  int
  file_open(const char *path, struct File **pf)
  ```

- 从文件f的偏移offset处读count个字节到buf处。这里要**注意**count和offset可能不是页对齐的，需要进行特殊处理

  操作就是通过`file_get_block`函数找到文件f的数据块的虚拟地址，然后将该虚拟地址的内容拷贝到buf处。如果发生页错误，那么会进行读磁盘。

  ```
  ssize_t
  file_read(struct File *f, void *buf, size_t count, off_t offset)
  ```

- 从buf处读count个字节到f的offset偏移处

  ```
  int
  file_write(struct File *f, const void *buf, size_t count, off_t offset)
  ```

- 将文件f的第filebno个块free掉。调用`free_block`，并且修改f中相关的块号为0

  ```
  static int
  file_free_block(struct File *f, uint32_t filebno)
  ```

- 按照newsize缩小文件f的大小，从文件末尾free掉(old_nblocks - new_nblocks)个磁盘块

  ```
  // 按照newsize缩小文件f的大小，从文件末尾free掉(old_nblocks - new_nblocks)个磁盘块
  static void
  file_truncate_blocks(struct File *f, off_t newsize)
  ```

- 将文件f**所有的脏页**都刷到磁盘。

  ```
  // 将文件f所有的脏页都刷到磁盘，
  // 1. 先刷其内容，也就是指向的块
  // 2. 然后刷文件f的信息块File
  // 3. 最后刷文件f的indirect块
  void
  file_flush(struct File *f)
  ```

- 刷新所有的磁盘

  ```
  // Sync the entire file system.  A big hammer.
  // 刷新所有的磁盘
  void
  fs_sync(void)
  ```

### 实现普通进程对文件的操作

- 以上实现的文件操作函数只能是FS进程调用，普通进程无法调用，甚至普通进程连*block cache*都没有
- 于是JOS就实现了普通进程通过IPC来与FS进程进行通信，进而让FS进程代替普通进程实现文件操作

#### Open

##### 普通进程

- JOS管理打开文件有两个数据结构，一个是FS进程中的`OpenFile`，这个数据结构在文件的*meta-data*`File`之上又添加了几个数据，比如在打开文件表中的id，该文件的文件描述符Fd。

  ```
  // 文件系统(fs)掌管着所有进程的打开文件，一个文件系统最多同时有1024个打开文件
  // fs的openFile里的o_fd和某一进程的Fd都映射到同一物理空间
  // 也就是进程的Fd都对应着fs里的某一个OpenFile
  struct OpenFile {
     uint32_t o_fileid; // file id
     struct File *o_file;   // mapped descriptor for open file
     int o_mode;       // open mode
     struct Fd *o_fd;   // Fd page
  };
  ```

- 文件的文件描述符在FS进程空间和普通进程空间各保存一份，即**同一个打开文件的Fd的物理地址**映射到FS进程空间和普通进程空间，引用数为2。文件描述符包含该文件的偏移，表示读或写从该文件的哪里开始。

  ```
  struct Fd {
     int fd_dev_id;
     off_t fd_offset;   // 初始化为0
     int fd_omode;
     union {
        // File server files
        struct FdFile fd_file;
     };
  };
  ```

- 以user/cat.c的代码为例，首先调用file.c下的Open函数，可知我们先在该进程内分配一个空闲的Fd，此时该Fd还没有任何内容（还未给该Fd分配物理内存），然后把FS调用打开文件函数所需的文件路径和文件open mode放到一个数据结构`Fsipc`中，然后调用`fsipc`与FS进程进行通信。通信完之后FS进程就给该进程刚分配的Fd分配一页数据，这样我们就得到了打开文件的Fd了。最后返回Fd在该进程中的序号。

  **$\textcolor{red}{注意:}$** 打开一个文件，那么FS进程就会给该文件分配一个`OpenFile`，因此该文件在打开文件表中也有序号。而普通进程一开始调用`fd_alloc`所"分配"的的Fd是在该进程的进程空间分配，**和FS进程的关系为**最终该进程的Fd的物理页和FS进程中所对应的`OpenFile`中`o_fd`的物理页相同。普通进程是无法访问`OpenFile`表的。

  ```
  int
  open(const char *path, int mode)
  {
     // Find an unused file descriptor page using fd_alloc.
     // Then send a file-open request to the file server.
     // Include 'path' and 'omode' in request,
     // and map the returned file descriptor page
     // at the appropriate fd address.
  
     int r;
     struct Fd *fd;
  
     if (strlen(path) >= MAXPATHLEN)
        return -E_BAD_PATH;
  
     if ((r = fd_alloc(&fd)) < 0)
        return r;
  
     strcpy(fsipcbuf.open.req_path, path);
     fsipcbuf.open.req_omode = mode;
  
     if ((r = fsipc(FSREQ_OPEN, fd)) < 0) {
          // 将fd free掉
        fd_close(fd, 0);
        return r;
     }
      // 最后得到的fd的(fd_file_id)为fs里OpenFile的id
  
     return fd2num(fd);
  }
  ```

- 普通进程和FS进程进行IPC的共享页为`Fsipc`结构的数据。根据不同的文件操作发送不同类型的数据。

  ```
  union Fsipc {
     struct Fsreq_open {
        char req_path[MAXPATHLEN];
        int req_omode;
     } open;
     struct Fsreq_read {
        int req_fileid;
        size_t req_n;
     } read;
     struct Fsret_read {
        char ret_buf[PGSIZE];
     } readRet;
     struct Fsreq_write {
        int req_fileid;
        size_t req_n;
        char req_buf[PGSIZE - (sizeof(int) + sizeof(size_t))];
     } write;
     ....
     ....
  
     // Ensure Fsipc is one page
     char _pad[PGSIZE];
  };
  ```

- 普通进程与FS进程进行交互的函数为`fsipc`，`type`告诉FS进程所要请求的操作类型，`fsipcbuf`为操作所需信息。在Open操作中，我们需要获得FS所分配的Fd物理内存，所以这里的`dstva`为请求进程的Fd虚拟地址。

  然后ipc_send发送请求，最后ipc_recv等待FS进程返回。

  ```
  // type: request code, passed as the simple integer IPC value.
  // dstva: virtual address at which to receive reply page, 0 if none.
  // Returns result from the file server.
  static int
  fsipc(unsigned type, void *dstva)
  {
     static envid_t fsenv;
     if (fsenv == 0)
        fsenv = ipc_find_env(ENV_TYPE_FS);
  
     static_assert(sizeof(fsipcbuf) == PGSIZE);
  
     if (debug)
        cprintf("[%08x] fsipc %d %08x\n", thisenv->env_id, type, *(uint32_t *)&fsipcbuf);
  
     ipc_send(fsenv, type, &fsipcbuf, PTE_P | PTE_W | PTE_U);
     return ipc_recv(NULL, dstva, NULL);
  }
  ```

##### FS进程

- 普通进程调用ipc_recv后就阻塞了，等待FS进程进行唤醒

- FS进程这边调用serve函数不断第接收普通进程的请求。FS进程使用`fsreq`来作为共享页获取请求内容。然后根据请求类型调用不同的服务函数。

  处理完后将相关信息发送给请求进程。Open操作中发送的是打开文件的Fd页。

  ```
  void
  serve(void)
  {
     uint32_t req, whom;
     int perm, r;
     void *pg;
  
     while (1) {
        perm = 0;
        req = ipc_recv((int32_t *) &whom, fsreq, &perm);
        if (debug)
           cprintf("fs req %d from %08x [page %08x: %s]\n",
              req, whom, uvpt[PGNUM(fsreq)], fsreq);
  
        // All requests must contain an argument page
        if (!(perm & PTE_P)) {
           cprintf("Invalid request from %08x: no argument page\n",
              whom);
           continue; // just leave it hanging...
        }
  
        pg = NULL;
        if (req == FSREQ_OPEN) {
           r = serve_open(whom, (struct Fsreq_open*)fsreq, &pg, &perm);
        } else if (req < ARRAY_SIZE(handlers) && handlers[req]) {
           r = handlers[req](whom, fsreq);
        } else {
           cprintf("Invalid request code %d from %08x\n", req, whom);
           r = -E_INVAL;
        }
        ipc_send(whom, r, pg, perm);
        sys_page_unmap(0, fsreq);
     }
  }
  ```

- 在open服务函数中，首先调用`openfile_alloc`函数分配一个空闲的`OpenFile`。

  判断空闲的方法为：查看该`OpenFile`中Fd所在的物理页的引用次数，如果为0或1，那么就是空闲的

  然后调用`file_open`函数获得该文件的`File`，并且初始化该文件的`OpenFile`。

  ```
  int
  serve_open(envid_t envid, struct Fsreq_open *req,
        void **pg_store, int *perm_store)
  {
     char path[MAXPATHLEN];
     struct File *f;
     int fileid;
     int r;
     struct OpenFile *o;
  
     if (debug)
        cprintf("serve_open %08x %s 0x%x\n", envid, req->req_path, req->req_omode);
  
     // Copy in the path, making sure it's null-terminated
     memmove(path, req->req_path, MAXPATHLEN);
     path[MAXPATHLEN-1] = 0;
  
     // Find an open file ID
     if ((r = openfile_alloc(&o)) < 0) {
        if (debug)
           cprintf("openfile_alloc failed: %e", r);
        return r;
     }
     fileid = r;
  
     // Open the file
     if (req->req_omode & O_CREAT) {
        if ((r = file_create(path, &f)) < 0) {
           if (!(req->req_omode & O_EXCL) && r == -E_FILE_EXISTS)
              goto try_open;
           if (debug)
              cprintf("file_create failed: %e", r);
           return r;
        }
     } else {
  try_open:
        if ((r = file_open(path, &f)) < 0) {
           if (debug)
              cprintf("file_open failed: %e", r);
           return r;
        }
     }
  
     // Truncate
     if (req->req_omode & O_TRUNC) {
        if ((r = file_set_size(f, 0)) < 0) {
           if (debug)
              cprintf("file_set_size failed: %e", r);
           return r;
        }
     }
     if ((r = file_open(path, &f)) < 0) {
        if (debug)
           cprintf("file_open failed: %e", r);
        return r;
     }
  
     // Save the file pointer
     o->o_file = f;
  
     // Fill out the Fd structure
     o->o_fd->fd_file.id = o->o_fileid;
     o->o_fd->fd_omode = req->req_omode & O_ACCMODE;
     o->o_fd->fd_dev_id = devfile.dev_id;
     o->o_mode = req->req_omode;
  
     if (debug)
        cprintf("sending success, page %08x\n", (uintptr_t) o->o_fd);
  
     // Share the FD page with the caller by setting *pg_store,
     // store its permission in *perm_store
     *pg_store = o->o_fd;
     *perm_store = PTE_P|PTE_U|PTE_W|PTE_SHARE;
  
     return 0;
  }
  ```

  ```
  // Allocate an open file.
  int
  openfile_alloc(struct OpenFile **o)
  {
     int i, r;
  
     // Find an available open-file table entry
     for (i = 0; i < MAXOPEN; i++) {
          // pageref: 看opentab[i].o_fd所在的页有多少映射
        switch (pageref(opentab[i].o_fd)) {
        case 0:
              // 如果有空余的打开文件槽，那么给该打开文件的FD分配一页，此时该页的ref为1，注意这里没有break
              // 所以之后会执行 case 1
           if ((r = sys_page_alloc(0, opentab[i].o_fd, PTE_P|PTE_U|PTE_W)) < 0)
              return r;
           /* fall through */
        case 1:
              // pageref(opentab[i].o_fd) 可能是0，1，2
              // 为0，说明这个open file为空闲的
              // 为1，说明这个open file被分配过，但是该进程把文件close了，所以ref就从2变为了1，这样又可以继续映射了
              // 为2，说明fs的open file里的o_fd和某一进程的Fd 都映射到这一页，所以引用数为2
           opentab[i].o_fileid += MAXOPEN;
           *o = &opentab[i];
           memset(opentab[i].o_fd, 0, PGSIZE);
           return (*o)->o_fileid;
        }
     }
     return -E_MAX_OPEN;
  }
  ```

#### Read

- 请求进程首先调用lib/fd.c中的`read`函数，由于我们读取的是磁盘上的文件，所以进而调用file.c里的`devfile_read`函数。

- 与Open不同的是，这里的`fsipcbuf`既做请求的共享页，又做FS进程返回的共享页，FS进程将内容读到`fsipcbuf`的物理内存中，最后请求进程将`fsipcbuf`里的内容复制到buf中。

  ```
  static ssize_t
  devfile_read(struct Fd *fd, void *buf, size_t n)
  {
     // Make an FSREQ_READ request to the file system server after
     // filling fsipcbuf.read with the request arguments.  The
     // bytes read will be written back to fsipcbuf by the file
     // system server.
     int r;
  
     fsipcbuf.read.req_fileid = fd->fd_file.id;
     fsipcbuf.read.req_n = n;
     if ((r = fsipc(FSREQ_READ, NULL)) < 0)
        return r;
     assert(r <= n);
     assert(r <= PGSIZE);
     memmove(buf, fsipcbuf.readRet.ret_buf, r);
     return r;
  }
  ```

- read服务函数通过调用`file_read`来读取。并且修改文件的offset，以便下一次接着读

  ```
  int
  serve_read(envid_t envid, union Fsipc *ipc)
  {
     struct Fsreq_read *req = &ipc->read;
     struct Fsret_read *ret = &ipc->readRet;
  
     if (debug)
        cprintf("serve_read %08x %08x %08x\n", envid, req->req_fileid, req->req_n);
  
     // Lab 5: Your code here:
      struct OpenFile *o;
      int r;
      if ((r = openfile_lookup(envid, req->req_fileid, &o)) < 0)
          return r;
      if ((r = file_read(o->o_file, ret->ret_buf, req->req_n, o->o_fd->fd_offset)) < 0) // an OpenFile has a fd!
          return r;
      o->o_fd->fd_offset += r;
      return r;
  }
  ```

#### Write

- 将buf中的内容写到fd所对应的文件中，所以这里的共享页`fsipcbuf`里的内容是要写入的内容

  ```
  static ssize_t
  devfile_write(struct Fd *fd, const void *buf, size_t n)
  {
     // Make an FSREQ_WRITE request to the file system server.  Be
     // careful: fsipcbuf.write.req_buf is only so large, but
     // remember that write is always allowed to write *fewer*
     // bytes than requested.
     // LAB 5: Your code here
      memmove(fsipcbuf.write.req_buf, buf, n);
      fsipcbuf.write.req_n = n;
      fsipcbuf.write.req_fileid = fd->fd_file.id;
      int r;
      if ((r = fsipc(FSREQ_WRITE, NULL)) < 0)
          return r;
      return n;
  }
  ```

- 使用`file_write`函数写入。

  ```
  int
  serve_write(envid_t envid, struct Fsreq_write *req)
  {
     if (debug)
        cprintf("serve_write %08x %08x %08x\n", envid, req->req_fileid, req->req_n);
  
     // LAB 5: Your code here.
      struct OpenFile *o;
      int r;
      if ((r = openfile_lookup(envid, req->req_fileid, &o)) < 0)
          return r;
      if ((r = file_write(o->o_file,  req->req_buf, req->req_n, o->o_fd->fd_offset)) < 0)
          return r;
      o->o_fd->fd_offset += r;
      return r;
  }
  ```

## Spawning Processes

- 接下来我们使用上面实现的文件系统来实现spawn

- spawn约等于fork + exec，也就是创建一个子进程，并且让子进程执行指定的程序

- prog为可执行文件的文件名，argv为该可执行文件入口函数的命令行参数，由于是多个参数，所以argv为 char **类型

  1. 按照文件名打开文件，得到Fd
  2. 该文件为Elf格式，读出*Elf Header*
  3. 使用`sys_exofork`创建子进程
  4. 初始化子进程的eip为可执行文件的入口地址
  5. 使用`init_stack`初始化子进程的用户栈，主要是将可执行文件的命令行参数入栈，初始化esp
  6. 利用elf 找到*program header* `ph`，然后依照`ph`提供的该段的虚拟地址、该段的大小、该段在可执行文件中的偏移调用`map_segment`将磁盘中的数据段读到虚拟地址处
  7. `copy_shared_pages`函数将父进程的Fd共享给子进程
  8. `sys_env_set_trapframe`开启子进程的外部中断，关闭子进程的IO权限
  9. 最后`sys_env_set_status`设置子进程的状态为就绪态

  ```
  int
  spawn(const char *prog, const char **argv)
  {
     unsigned char elf_buf[512];
     struct Trapframe child_tf;
     envid_t child;
  
     int fd, i, r;
     struct Elf *elf;
     struct Proghdr *ph;
     int perm;
  
      // 打开文件，得到Fd编号r
     if ((r = open(prog, O_RDONLY)) < 0)
        return r;
     fd = r;
  
     // Read elf header
     elf = (struct Elf*) elf_buf;
     if (readn(fd, elf_buf, sizeof(elf_buf)) != sizeof(elf_buf)
         || elf->e_magic != ELF_MAGIC) {
        close(fd);
        cprintf("elf magic %08x want %08x\n", elf->e_magic, ELF_MAGIC);
        return -E_NOT_EXEC;
     }
  
     // Create new child environment
     if ((r = sys_exofork()) < 0)
        return r;
     child = r;   // 子进程id
  
     // Set up trap frame, including initial stack.
      // 让子进程去执行新的程序：设置子进程的eip为该程序的入口地址
     child_tf = envs[ENVX(child)].env_tf;
     child_tf.tf_eip = elf->e_entry;
  
      // 给子进程的用户栈分配空间，并且初始化用户栈（执行程序所需的argc和argv参数入栈），初始化子进程的esp
     if ((r = init_stack(child, argv, &child_tf.tf_esp)) < 0)
        return r;
  
     // Set up program segments as defined in ELF header.
     ph = (struct Proghdr*) (elf_buf + elf->e_phoff);
     for (i = 0; i < elf->e_phnum; i++, ph++) {
        if (ph->p_type != ELF_PROG_LOAD)
           continue;
        perm = PTE_P | PTE_U;
        if (ph->p_flags & ELF_PROG_FLAG_WRITE)
           perm |= PTE_W;
        if ((r = map_segment(child, ph->p_va, ph->p_memsz,
                   fd, ph->p_filesz, ph->p_offset, perm)) < 0)
           goto error;
     }
     close(fd);
     fd = -1;
  
     // Copy shared library state.
     if ((r = copy_shared_pages(child)) < 0)
        panic("copy_shared_pages: %e", r);
  
     child_tf.tf_eflags |= FL_IOPL_3;   // devious: see user/faultio.c   ???
     if ((r = sys_env_set_trapframe(child, &child_tf)) < 0)
        panic("sys_env_set_trapframe: %e", r);
  
     if ((r = sys_env_set_status(child, ENV_RUNNABLE)) < 0)
        panic("sys_env_set_status: %e", r);
  
     return child;
  
  error:
     sys_env_destroy(child);
     close(fd);
     return r;
  }
  ```

- `map_segment`函数是将偏移量为fileoffset的fd文件大小为memsz的数据读到子进程va处

  调用该函数的为父进程

  1. 首先将内容读到父进程的`UTEMP`处
  2. 然后将子进程的va映射到父进程`UTEMP`相同的物理地址
  3. unmap父进程的`UTEMP`

  ```
  static int
  map_segment(envid_t child, uintptr_t va, size_t memsz,
     int fd, size_t filesz, off_t fileoffset, int perm)
  {
     int i, r;
     void *blk;
  
     //cprintf("map_segment %x+%x\n", va, memsz);
  
      // 保证页对齐
     if ((i = PGOFF(va))) {
        va -= i;
        memsz += i;
        filesz += i;
        fileoffset -= i;
     }
  
     for (i = 0; i < memsz; i += PGSIZE) {
        if (i >= filesz) {
           // allocate a blank page
           if ((r = sys_page_alloc(child, (void*) (va + i), perm)) < 0)
              return r;
        } else {
           // from file
           if ((r = sys_page_alloc(0, UTEMP, PTE_P|PTE_U|PTE_W)) < 0)
              return r;
           if ((r = seek(fd, fileoffset + i)) < 0)
              return r;
           if ((r = readn(fd, UTEMP, MIN(PGSIZE, filesz-i))) < 0)
              return r;
           if ((r = sys_page_map(0, UTEMP, child, (void*) (va + i), perm)) < 0)
              panic("spawn: sys_page_map data: %e", r);
           sys_page_unmap(0, UTEMP);
        }
     }
     return 0;
  }
  ```

- `init_stack`初始化子进程栈。

  布局为：参数字符串、指向第i个字符串的指针argv_store[i]、指向argv_store的指针argv、argc

  其中argv为char **类型，argc为int类型，argv_store为char *

  ```
  static int
  init_stack(envid_t child, const char **argv, uintptr_t *init_esp)
  {
     size_t string_size;
     int argc, i, r;
     char *string_store;
     uintptr_t *argv_store;
  
     // Count the number of arguments (argc)
     // and the total amount of space needed for strings (string_size).
     string_size = 0;
     for (argc = 0; argv[argc] != 0; argc++)
        string_size += strlen(argv[argc]) + 1;
  
     // Determine where to place the strings and the argv array.
     // Set up pointers into the temporary page 'UTEMP'; we'll map a page
     // there later, then remap that page into the child environment
     // at (USTACKTOP - PGSIZE).
     // strings is the topmost thing on the stack.
     string_store = (char*) UTEMP + PGSIZE - string_size;
     // argv is below that.  There's one argument pointer per argument, plus
     // a null pointer.
     argv_store = (uintptr_t*) (ROUNDDOWN(string_store, 4) - 4 * (argc + 1));
  
     // Make sure that argv, strings, and the 2 words that hold 'argc'
     // and 'argv' themselves will all fit in a single stack page.
     if ((void*) (argv_store - 2) < (void*) UTEMP)
        return -E_NO_MEM;
  
     // Allocate the single stack page at UTEMP.
     if ((r = sys_page_alloc(0, (void*) UTEMP, PTE_P|PTE_U|PTE_W)) < 0)
        return r;
  
     for (i = 0; i < argc; i++) {
          argv_store[i] = UTEMP2USTACK(string_store);
          strcpy(string_store, argv[i]);
          string_store += strlen(argv[i]) + 1;
      }
     argv_store[argc] = 0;
     assert(string_store == (char*)UTEMP + PGSIZE);
  
     argv_store[-1] = UTEMP2USTACK(argv_store);
     argv_store[-2] = argc;
  
     *init_esp = UTEMP2USTACK(&argv_store[-2]);
  
     // After completing the stack, map it into the child's address space
     // and unmap it from ours!
     if ((r = sys_page_map(0, UTEMP, child, (void*) (USTACKTOP - PGSIZE), PTE_P | PTE_U | PTE_W)) < 0)
        goto error;
     if ((r = sys_page_unmap(0, UTEMP)) < 0)
        goto error;
  
     return 0;
  
  error:
     sys_page_unmap(0, UTEMP);
     return r;
  }
  ```
