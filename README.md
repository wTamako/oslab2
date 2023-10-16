一、	练习1
1.	default_init函数：
•	作用：初始化物理内存管理器，具体是初始化空闲块列表（free_list）和空闲块数量（nr_free）。
•	过程：使用list_init初始化空闲块列表，将空闲块数量设为0。
•	解释：
ist_init(&free_list);：使用 list.h 中的 list_init 函数初始化空闲块列表，确保列表为空。
nr_free = 0;：将空闲块数量初始化为零，因为刚开始时没有可用的空闲块。
这个函数在物理内存管理器初始化阶段调用，确保管理器的状态是初始的、空闲块列表为空，并且空闲块数量为零。

2.	default_init_memmap函数：
•	作用：初始化内存映射，将一块空闲内存初始化为可用的内存块。
•	过程：
o	遍历指定数量的页面，初始化每个页面的属性，包括标记为非保留（PageReserved）和清零引用计数。
o	设置第一个页面的属性为可用内存块的总数，标记为已分配（SetPageProperty）。
o	将这些页面加入空闲块列表（free_list）中，并更新空闲块数量（nr_free）。
•	解释：
    assert(n > 0);：确保要初始化的页面数量 n 大于零。
    struct Page *p = base; for (; p != base + n; p ++) { ... }：遍历指定数量的页面。
    assert(PageReserved(p));：确保页面 p 是被保留的（reserved），即未被使用的空闲页面。
    p->flags = p->property = 0;：将页面的标志和属性清零，表示该页面是可用的。
    set_page_ref(p, 0);：将页面的引用计数设置为零，因为此时页面是空闲的，没有被引用。
    base->property = n;：将第一个页面的属性设置为可用内存块的总数。
    SetPageProperty(base);：标记第一个页面为已分配。
    nr_free += n;：增加空闲块数量。
    if (list_empty(&free_list)) { ... } else { ... }：根据空闲块列表是否为空进行不同的处理。
        如果空闲块列表为空，直接将第一个页面添加到空闲块列表中。
        如果空闲块列表不为空，遍历空闲块列表，找到正确的位置插入第一个页面。确保列表按地址从低到高有序。

这个函数在初始化物理内存时调用，将一块连续的空闲内存划分为可用的内存块，并更新空闲块列表和数量。

3.	default_alloc_pages函数：
•	作用：根据首次适应算法分配一定数量的页面。
•	过程：
o	遍历空闲块列表，找到第一个大小大于等于需求的空闲块。
o	如果找到，将该块分配给请求，可能需要将剩余部分作为新的空闲块加入列表。
o	更新空闲块数量，标记已分配的页面。
o	返回分配的页面地址。
•	解释：
    assert(n > 0);：确保要分配的页面数量 n 大于零。
    if (n > nr_free) { return NULL; }：如果需要分配的页面数量大于当前的空闲页面数量，则无法分配，返回 NULL。
    list_entry_t *le = &free_list; while ((le = list_next(le)) != &free_list) { ... }：遍历空闲块列表，寻找第一个满足分配条件的页面。
    struct Page *p = le2page(le, page_link); if (p->property >= n) { ... }：找到了满足分配条件的页面。
        如果该页面的空闲块数量大于等于需要分配的数量，则可以直接使用该页面。
    list_entry_t* prev = list_prev(&(page->page_link)); list_del(&(page->page_link));：从空闲块列表中删除该页面。
    if (page->property > n) { ... }：如果该页面的空闲块数量大于分配的数量，需要将剩余的部分重新加入空闲块列表。
        struct Page *p = page + n; p->property = page->property - n; SetPageProperty(p); list_add(prev, &(p->page_link));：将剩余的部分标记为已分配，并加入空闲块列表。
    nr_free -= n;：更新空闲页面数量。
    ClearPageProperty(page);：将已分配的页面标记为未分配。
    return page;：返回分配的页面。

这个函数实现了在空闲块列表中查找并分配指定数量的页面，并对空闲块列表进行相应的更新。
•  default_free_pages函数：
•	作用：释放一定数量的页面，可能合并相邻的空闲块。
•	过程：
o	遍历要释放的页面，清零页面属性，增加引用计数。
o	将这些页面加入空闲块列表中，可能会合并相邻的空闲块。
o	更新空闲块数量。
•	解释：
    assert(n > 0);：确保要释放的页面数量 n 大于零。
    struct Page *p = base; for (; p != base + n; p ++) { assert(!PageReserved(p) && !PageProperty(p)); p->flags = 0; set_page_ref(p, 0); }：循环遍历要释放的页面，确保这些页面既不是保留页面也不是已分配页面，并将其标记为未分配。
    base->property = n; SetPageProperty(base); nr_free += n;：更新要释放的页面的属性，标记为已分配，更新空闲页面数量。
    如果空闲块列表为空，或者要释放的页面的基地址小于列表中的第一个页面的基地址，将要释放的页面插入到空闲块列表的最前面。
    否则，遍历空闲块列表找到适当的位置插入要释放的页面，保持列表的有序性。
    list_entry_t* le = list_prev(&(base->page_link)); if (le != &free_list) { ... }：在空闲块列表中找到当前页面的前一个页面。
        p = le2page(le, page_link); if (p + p->property == base) { ... }：如果前一个页面的结束地址与当前页面的基地址相同，说明可以将它们合并成一个更大的空闲块。
            p->property += base->property; ClearPageProperty(base); list_del(&(base->page_link)); base = p;：合并空闲块，更新属性，从列表中删除当前页面。
    le = list_next(&(base->page_link)); if (le != &free_list) { ... }：在空闲块列表中找到当前页面的后一个页面。
        p = le2page(le, page_link); if (base + base->property == p) { ... }：如果后一个页面的起始地址与当前页面的结束地址相同，说明可以将它们合并成一个更大的空闲块。
            base->property += p->property; ClearPageProperty(p); list_del(&(p->page_link));：合并空闲块，更新属性，从列表中删除后一个页面。

这个函数实现了释放一定数量的页面，并在空闲块列表中更新，可能会合并相邻的空闲块。
•  default_nr_free_pages函数：
•	作用：返回当前空闲页面的数量。

4.	优化
内存碎片问题： First Fit 可能会产生外部碎片，即已分配的内存块之间存在一些不可用的小块。可以考虑其他更高级的分配算法，如 Best Fit，以最小化碎片。
分区的动态调整： 在 First Fit 中，空闲块列表通常按照地址排序，但这可能导致频繁的块分裂和合并。可以考虑实现一种动态调整分区大小的策略，以更好地满足不同大小的内存请求。

二、	练习2
1.	初始化空闲页
best_fit_init_memmap接收一个物理页的地址Page* base和一个size_t n作为参数，用于将一段以base为基地址的大小为n的连续物理页初始化为空闲页面，其实现和default_init_memmap是一样的：
遍历要初始化的这段物理页，清空它们的标志和属性信息，设置其引用计数为0，并将base的属性设为n。
根据base的值按照由小到大的顺序（相当于物理页的地址从低到高）将其链入free_list中，即遍历free_list，若找到第一个大于base的页，将base插入到它前面，若已经到达链表结尾，将base插入到链表尾部。
2.	物理内存的分配和释放
best_fit_alloc_pages接收一个参数size_t n，用于分配一段大小为n的连续物理页，其与First Fit算法的分配方式的不同之处在于：在遍历空闲链表的过程中，如果找到满足需求的空闲页框，它会记录该页面以及当前找到的最小连续空闲页框数量，从而保证所找到的空闲页数量是和要分配的物理页数量n最为接近。
其余的操作和First Fit的一致，包括从空闲链表中删除分配出去的页面，然后更新page + n处的空闲页属性，并将其链入前面的空闲页之后，将空闲页nr_free的数量减n，以及清除掉分配出去的空闲页的属性信息等。

best_fit_free_pages接收一个物理页的地址Page* base和一个size_t n作为参数，用于释放一段分配出去的以base为基地址的大小为n的连续物理页，其实现和default_free_pages也是一样的：
遍历要释放的这段物理页，清空它们的标志位，设置其引用计数为0。
将base的属性设置为释放的页块数，并将其标记为已分配状态，然后令nr_free的值增加n，再将base重新链入空闲链表。
将当前页与其前后页块合并，具体来说：先判断前面的空闲页块是否与当前页块连续，若连续，则将前一个空闲页块的大小加上当前页块的大小，清除当前页块的属性标记，再从链表中删除当前页块，然后将指针指向前一个空闲页块，以相同的方式将其与后续的页块合并。

3.	改进
Best Fit算法的性能可能较低，因为它需要遍历整个空闲内存块链表来找到合适的块。可以通过引入一些数据结构，如二叉树或其他索引结构，来提高查找速度。这样可以更快地找到合适大小的内存块，减少分配和释放的时间复杂度。

Best Fit算法通常不会动态调整内存块的大小，这可能导致不同大小的内存块之间的浪费。考虑实现一种机制，可以在运行时动态调整内存块的大小，以更好地适应实际分配和释放的情况。这可以通过合并相邻的小块或拆分大块来实现。

三、	
