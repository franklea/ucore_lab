【练习0】
请把你做的实验1的代码填入本实验中代码中有“LAB1”的注释相应部分。

【练习1】
实现first-fit连续物理内存分配算法。
该练习主要包括3个部分，初始化，分配，释放。
初始化：
实现在default_init_memmap函数中，如下：
'''
static void default_init_memmap(struct Page *base, size_t n) 
{
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));
        p->flags = PG_property;
        SetPageProperty(p);
        p->property = 0;
        set_page_ref(p, 0);
        list_add_before(&free_list,&(p->page_link));
    }
    base->property = n;
    nr_free += n;
}
'''
从存储的起始地址开始，分别对每一页进行初始化，主要操作包括：
p->flags置为PG_property，表示该页可用；
块中第一个空闲页的property置为n,其他页的property置为0;
p->ref也置为0，因为当前p是free的，还没有被引用。
使用p->page_link将这些页连接到free_list中。
最后，设置free mem block的总数nr_free:nr_free += n;

分配：
实现在default_alloc_pages函数中，根据first-fit原理，即找到第一个满足需求的block，分配，然后将该block剩下的部分加入空闲列表中。主要实现步骤如下：
1)遍历free_list中的块，取出每个块的第一个页，比较该页的property（即该块的大小）是否不小于分配需求n：
'''
  while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        if (p->property >= n) {
'''
若满足，则继续下面操作：
2）将该块的前n页分配，其中需要对这些页的一些标志位进行设置，具体有：
pg->reserved =1;
pg->property = 0;
接着从free_list中删除这些页，实现时，先删除该块，然后将该块的剩余部分组成另一个块再添加到free_list中。
	'''
	for (i=0; i < n; i++){
                len = list_next(le);
                struct Page *pp = le2page(le, page_link);
                SetPageReserved(pp);
                ClearPageProperty(pp);
                list_del(le);
                le = len;
            }

            if (p->property > n){
                (le2page(le, page_link))->property = p->property - n;
            }

	'''
3）最后重新计算free_list的总量:nr_free -= n;
4）若分配成功返回分配的内存p，否则，返回空。

释放：
是现在default_free_pages函数中，传递的参数为要释放的块起始地址以及块的大小。具体实现如下：
1）遍历free_list，根据地址有序，找到要插入的位置。
'''
//find the correct position     
        while ((le=list_next(le))!= &free_list) {
                p = le2page(le,page_link);
                if (p > base){
                        break;
                }
        }
'''
2）按照地址从低到高的顺序，插入各个页：
'''
//from low to high address, insert the pages.
        for(p = base; p < base+n; p++)
        {
                list_add_before(le, &(p->page_link));
        }
'''
3)尝试合并，分别查看free_list中前面和后面的块，看是否刚好存在可以合并的块，即两个块的地址可以连续。
'''
//try to merge low or high addr blocks
        p = le2page(le,page_link);
        if (base+n == p){
                base->property += p->property;
                p->property = 0;
        }

        le = list_prev(&(base->page_link));
        p = le2page(le, page_link);
        if(le != &free_list && p == base-1){
                while( le != &free_list){
                        if (p->property){
                                p->property += base->property;
                                base->property = 0;
                                break;
                        }
                        le = list_prev(le);
                        p = le2page(le, page_link);
                }
        }

'''
4)修改free_list总量：nr_free += n
你的first fit算法是否有进一步改进的空间？

【练习2】实现寻找虚拟地址对应的页表项实现：
'''
 pde_t *pde = &pgdir[PDX(la)];
    if (!(*pde & PTE_P)) {
        struct Page* page = NULL;
        if (create && (page = alloc_page()) != NULL) {
            set_page_ref(page, 1);
            uintptr_t pa = page2pa(page);
            memset(KADDR(pa), 0, PGSIZE);
            *pde = pa | PTE_P | PTE_W | PTE_U;
        }
        else{
            return NULL;
        }
    }
    return &((pte_t *)KADDR((*pde)&0xFFFFF000))[PTX(la)];

'''
首先回去一级页目录入口，若存在，则看是否需要创建新的二级也表，若不需要或者创建失败，则返回NULL；若创建成功，则对该页表进行相应的设置，分别是设定引用数，初始化页表内容，再在一级页表中创建页目录项指向表示二级页表的新物理页。
请描述页目录项(Page Director Entry)和页表(Page Table Entry)中每个组成部分的含义和以及对ucore而言的潜在用处。
页目录项：
页目录项内容 = 页表起始物理地址	|	PTE_U	|	PTE_W	|	PTE_P
页表项：
页表项内容 = (pa & 0x0FFF)	|	PTE_P	|	PTE_W
其中:
	 PTE_U:位3,表示用户态的软件可以读取对应地址的物理内存页内容
	 PTE_W:位2,表示物理内存页内容可写
	 PTE_P:位1,表示物理内存页存在
潜在用处，可用于内存共享时的权限控制。
如果ucore执行过程中访问内存,出现了页访问异常,请问硬件要做哪些事情?
会产生pagefault中断，并进行相应的缺页中断处理，如访问磁盘，更新缓存，页表等。


【练习3】释放某虚地址所在的页并取消对应二级页表项的映射
实现如下：
'''
if (*ptep & PTE_P) {
                struct Page *page = pte2page(*ptep);
                if (page_ref_dec(page) == 0) {
                        free_page(page);
                }
                *ptep = 0;
                tlb_invalidate(pgdir, la);
        }

'''
若物理页存在，找到对应的页，若该页没有额外的引用，则释放；然后使快表中对应的表项失效。

数据结构Page的全局变量(其实是一个数组)的每一项与页表中的页目录项和页表项有无对应关系?如果有,其对应关
系是啥?
有，Page变量中是页的虚拟地址。
physical_address == &pages[PPN(pa)], physical_address == virtual_address - KERNBASE

