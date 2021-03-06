The hugetlb analyze based on kernel 4.1.21


# The hugetlb key structure:
```c

super_block
{
...
	void *s_fs_info  /* Filesystem private info */
		|
		|
		 \___ >>> hugetlbfs_sb_info
		     {
		      ...
			struct hstate *hstate /* Defines one hugetlb page size */
			{
				int next_nid_to_alloc;
				int next_nid_to_free;
				unsigned int order;
				unsigned long mask;
				unsigned long max_huge_pages;
				unsigned long nr_huge_pages;
				unsigned long free_huge_pages;
				unsigned long resv_huge_pages;
				unsigned long surplus_huge_pages;
				unsigned long nr_overcommit_huge_pages;
				struct list_head hugepage_activelist;
				struct list_head hugepage_freelists[MAX_NUMNODES];
				unsigned int nr_huge_pages_node[MAX_NUMNODES];
				unsigned int free_huge_pages_node[MAX_NUMNODES];
				unsigned int surplus_huge_pages_node[MAX_NUMNODES];
			#ifdef CONFIG_CGROUP_HUGETLB
				/* cgroup control files */
				struct cftype cgroup_files[5];
			#endif
				char name[HSTATE_NAME_LEN];
			}
		      ...

		     }
...
}

```
***
This is the type of page, in order to void the **memory fragmentation**
***
```c
enum {
        MIGRATE_UNMOVABLE,
        MIGRATE_RECLAIMABLE,
        MIGRATE_MOVABLE,
        MIGRATE_PCPTYPES,       /* the number of types on the pcp lists */
        MIGRATE_RESERVE = MIGRATE_PCPTYPES,
#ifdef CONFIG_CMA                               
        /*
         * MIGRATE_CMA migration type is designed to mimic the way
         * ZONE_MOVABLE works.  Only movable pages can be allocated
         * from MIGRATE_CMA pageblocks and page allocator never
         * implicitly change migration type of MIGRATE_CMA pageblock.
         *
         * The way to use it is to change migratetype of a range of
         * pageblocks to MIGRATE_CMA which can be done by
         * __free_pageblock_cma() function.  What is important though
         * is that a range of pageblocks must be aligned to
         * MAX_ORDER_NR_PAGES should biggest page be bigger then
         * a single pageblock.
         */
        MIGRATE_CMA,
#endif
#ifdef CONFIG_MEMORY_ISOLATION
        MIGRATE_ISOLATE,        /* can't allocate from here */
#endif
        MIGRATE_TYPES
};

```
***
This is the fallback of free_list[MIGRATE_TYPES], when free_list[xxx] are not enough, then gather the momery from the fallback[][]
***
```c
/*
 * This array describes the order lists are fallen back to when
 * the free lists for the desirable migrate type are depleted
 */
static int fallbacks[MIGRATE_TYPES][4] = {
        [MIGRATE_UNMOVABLE]   = { MIGRATE_RECLAIMABLE, MIGRATE_MOVABLE,     MIGRATE_RESERVE },
        [MIGRATE_RECLAIMABLE] = { MIGRATE_UNMOVABLE,   MIGRATE_MOVABLE,     MIGRATE_RESERVE },
        [MIGRATE_MOVABLE]     = { MIGRATE_RECLAIMABLE, MIGRATE_UNMOVABLE,   MIGRATE_RESERVE },
#ifdef CONFIG_CMA
        [MIGRATE_CMA]         = { MIGRATE_RESERVE }, /* Never used */
#endif
        [MIGRATE_RESERVE]     = { MIGRATE_RESERVE }, /* Never used */
#ifdef CONFIG_MEMORY_ISOLATION
        [MIGRATE_ISOLATE]     = { MIGRATE_RESERVE }, /* Never used */
#endif
};

/* If huge pages are not used, group by MAX_ORDER_NR_PAGES */
#define pageblock_order         (MAX_ORDER-1)

#endif /* CONFIG_HUGETLB_PAGE */

#define pageblock_nr_pages      (1UL << pageblock_order)

```
# Hugetlb do_page_fault

Hugetlb allocate memory via do_page_fault() function, it is why called hugetlb as a patch of MM.
```c
COW:

do_page_fault		      ----|
  __do_page_fault		  |>( all of these operations are standerd do_page_fault path )
   handle_mm_fault		  |
    __handle_mm_fault         ----|
     hugetlb_fault  	      ====> ( At this point, the allocate page go through hugetlb path )
      hugetlb_no_page
       alloc_huge_page
	dequeue_huge_page_vma ====> ( get page from hugepage_freelists[] )
	alloc_buddy_huge_page ====> ( get page via buddy system )
	  alloc_pages	      ----> ( At this point, the allocate page get back to standerd alloc_pages() )
	    alloc_pages_node
	      __alloc_pages
		__alloc_pages_nodemask   /* This is the 'heart' of the zoned buddy allocator */
		  get_page_from_freelist /* goes through the zonelist trying to allocate a page */
			 buffered_rmqueue /* allocate from buddy system */
		  __alloc_pages_slowpath /* No enough page in zonelist, try to allocate page from being preactive releasing some space */
```
***
elaborate the *__do_page_fault* before *handle_mm_fault*
***

Regarding requirement for do_page_fault, divede this operation into two parts:
1. Kernel mode.
2. User mode.

There is the basic overview of it:
![Alt text](/pic/do_page_fault_overview.png)

This is the detailed flow of *__do_page_fault* before *handle_mm_fault*

The detailed steps:

![Alt text](/pic/do_page_fault_detail.png)

```c
Only analyze 32-bit situation:
static void __kprobes __do_page_fault(struct pt_regs *regs, unsigned long write,
        unsigned long address)
{
...
/*
 * The first of all, begin to Check Kernel space
 *
 */

            /* If adress is during in VMALLOC_START and VMALLOC_END or
             * If adress is during in MODULE_START and MODULE_END.
             * Both of them all go to no_context
             * define VMALLOC_FAULT_TARGET no_context
             */
            if (unlikely(address >= VMALLOC_START && address <= VMALLOC_END))
                goto VMALLOC_FAULT_TARGET;
#ifdef MODULE_START
        if (unlikely(address >= MODULE_START && address < MODULE_END))
                goto VMALLOC_FAULT_TARGET;
#endif

...
/*
 * Secondly, Check the user space
 *
 */
        /*
         * If we're in an interrupt or have no user
         * context, we must not take the fault..
         */
        if (in_atomic() || !mm)
                goto bad_area_nosemaphore;

retry:
        down_read(&mm->mmap_sem);
        /* Get the vma via mm and address */
        vma = find_vma(mm, address);

        /* If vma is null, the adress is invalid, go to bad_area */
        if (!vma)
                goto bad_area;

        /* If address >= vm_start, the address is valid, go to good_area */
        if (vma->vm_start <= address)
                goto good_area;

        /* If vma->vm_flags == VM_GROWSDOWN, it means the adress stay in stack area, need to expand it, so go to next code for getting expand_stack */
        if (!(vma->vm_flags & VM_GROWSDOWN))
                goto bad_area;
        if (expand_stack(vma, address))
                goto bad_area;
...
good_area:

         info.si_code = SEGV_ACCERR;
         /* If write is valid, check vma->vm_flags == VM_WRITE? If Yes, Keep going, If No, go to bad_area */
        if (write) {
                if (!(vma->vm_flags & VM_WRITE))
                        goto bad_area;
                flags |= FAULT_FLAG_WRITE;
        } else {

                /*
                * #define MIPS_CPU_RIXI           0x00800000ull /* CPU has TLB Read/eXec Inhibit */
                * # define cpu_has_rixi           (cpu_data[0].options & MIPS_CPU_RIXI)
                * It means the TLB is inhibit
                */
                if (cpu_has_rixi) {
                        if (address == regs->cp0_epc && !(vma->vm_flags & VM_EXEC)) {
                                goto bad_area;
                        }

                        /* If read is valid, check vma->flags == VM_READ? If Yes, keep going, If No, go to bad_area */
                        if (!(vma->vm_flags & VM_READ)) {
                                goto bad_area;
                        }
                } else {

                /*
                 * TLB is not inhibit, no pages for read, so, one of the  VM_READ, VM_WRITE or VM_EXEC need to be set, otherwise, go to bad_area.
                 *
                 *
                */
                        if (!(vma->vm_flags & (VM_READ | VM_WRITE | VM_EXEC)))
                                goto bad_area;
                }
        }

        /*
         * If for any reason at all we couldn't handle the fault,
         * make sure we exit gracefully rather than endlessly redo
         * the fault.
         */
        fault = handle_mm_fault(mm, vma, address, flags);
...
bad_area:
        up_read(&mm->mmap_sem);

bad_area_nosemaphore:
        /* User mode accesses just cause a SIGSEGV */
        if (user_mode(regs)) {
...
                force_sig_info(SIGSEGV, &info, tsk);
                return;
        }

no_context:
        /* Are we prepared to handle this kernel fault?  */
        if (fixup_exception(regs)) {
                current->thread.cp0_baduaddr = address;
                return;
        }

        /*
         * Oops. The kernel tried to access some bad page. We'll have to
         * terminate things with extreme prejudice.
         */
        bust_spinlocks(1);

        printk(KERN_ALERT "CPU %d Unable to handle kernel paging request at "
               "virtual address %0*lx, epc == %0*lx, ra == %0*lx\n",
               raw_smp_processor_id(), field, address, field, regs->cp0_epc,
               field,  regs->regs[31]);
        die("Oops", regs);

out_of_memory:
        /*
         * We ran out of memory, call the OOM killer, and return the userspace
         * (which will retry the fault, or kill us if we got oom-killed).
         */
        up_read(&mm->mmap_sem);
        if (!user_mode(regs))
                goto no_context;
        pagefault_out_of_memory();
        return;

do_sigbus:
        up_read(&mm->mmap_sem);

        /* Kernel mode? Handle exceptions or die */
        if (!user_mode(regs))
                goto no_context;
        else
        /*
         * Send a sigbus, regardless of whether we were in kernel
         * or user mode.
         */
...
        force_sig_info(SIGBUS, &info, tsk);

        return;
...
}
```
***
handle_mm_fault
	__handle_mm_fault

**NOTE:** The main function of it is that create the "pud" via "pgd", and create "pmd" via "pud", during these operations, deal with the huge tlb via hugetlb_fault(); later, call handle_pte_fault to tackle with "pte"

***
```c
/*
 * By the time we get here, we already hold the mm semaphore
 *
 * The mmap_sem may have been released depending on flags and our
 * return value.  See filemap_fault() and __lock_page_or_retry().
 */
static int __handle_mm_fault(struct mm_struct *mm, struct vm_area_struct *vma,
                             unsigned long address, unsigned int flags)
{
        pgd_t *pgd;
        pud_t *pud;
        pmd_t *pmd;
        pte_t *pte;

        /* Deal with hugetlb_fault */
        if (unlikely(is_vm_hugetlb_page(vma)))
                return hugetlb_fault(mm, vma, address, flags);

        pgd = pgd_offset(mm, address);
        pud = pud_alloc(mm, pgd, address);
        if (!pud)
                return VM_FAULT_OOM;
        pmd = pmd_alloc(mm, pud, address);
        if (!pmd)
                return VM_FAULT_OOM;
        if (pmd_none(*pmd) && transparent_hugepage_enabled(vma)) {
                int ret = VM_FAULT_FALLBACK;
                if (!vma->vm_ops)
                        ret = do_huge_pmd_anonymous_page(mm, vma, address,
                                        pmd, flags);
                if (!(ret & VM_FAULT_FALLBACK))
                        return ret;
        } else {
                pmd_t orig_pmd = *pmd;
                int ret;

                barrier();
                if (pmd_trans_huge(orig_pmd)) {
                        unsigned int dirty = flags & FAULT_FLAG_WRITE;

                        /*
                         * If the pmd is splitting, return and retry the
                         * the fault.  Alternative: wait until the split
                         * is done, and goto retry.
                         */
                        if (pmd_trans_splitting(orig_pmd))
                                return 0;

                        if (pmd_protnone(orig_pmd))
                                return do_huge_pmd_numa_page(mm, vma, address,
                                                             orig_pmd, pmd);
                        if (dirty && !pmd_write(orig_pmd)) {
                                ret = do_huge_pmd_wp_page(mm, vma, address, pmd,
                                                          orig_pmd);
                                if (!(ret & VM_FAULT_FALLBACK))
                                        return ret;
                        } else {
                                huge_pmd_set_accessed(mm, vma, address, pmd,
                                                      orig_pmd, dirty);
                                return 0;
                        }
                }
        }

        /*
         * Use __pte_alloc instead of pte_alloc_map, because we can't
         * run pte_offset_map on the pmd, if an huge pmd could
         * materialize from under us from a different thread.
         */
        if (unlikely(pmd_none(*pmd)) &&
            unlikely(__pte_alloc(mm, vma, pmd, address)))
                return VM_FAULT_OOM;
        /*
         * If a huge pmd materialized under us just retry later.  Use
         * pmd_trans_unstable() instead of pmd_trans_huge() to ensure the pmd
         * didn't become pmd_trans_huge under us and then back to pmd_none, as
         * a result of MADV_DONTNEED running immediately after a huge pmd fault
         * in a different thread of this mm, in turn leading to a misleading
         * pmd_trans_huge() retval.  All we have to ensure is that it is a
         * regular pmd that we can walk with pte_offset_map() and we can do that
         * through an atomic read in C, which is what pmd_trans_unstable()
         * provides.
         */
        if (unlikely(pmd_trans_unstable(pmd)))
                return 0;
        /*
         * A regular pmd is established and it can't morph into a huge pmd
         * from under us anymore at this point because we hold the mmap_sem
         * read mode and khugepaged takes it in write mode. So now it's
         * safe to run pte_offset_map().
         */
        pte = pte_offset_map(pmd, address);

        return handle_pte_fault(mm, vma, address, pte, pmd, flags);
}

```
***
handle_mm_fault
	__handle_mm_fault
		handle_pte_fault

1.

***

![Alt text](/)

```c
/*
 * These routines also need to handle stuff like marking pages dirty
 * and/or accessed for architectures that don't do it in hardware (most
 * RISC architectures).  The early dirtying is also good on the i386.
 *
 * There is also a hook called "update_mmu_cache()" that architectures
 * with external mmu caches can use to update those (ie the Sparc or
 * PowerPC hashed page tables that act as extended TLBs).
 *
 * We enter with non-exclusive mmap_sem (to exclude vma changes,
 * but allow concurrent faults), and pte mapped but not yet locked.
 * We return with pte unmapped and unlocked.
 *
 * The mmap_sem may have been released depending on flags and our
 * return value.  See filemap_fault() and __lock_page_or_retry().
 */
static int handle_pte_fault(struct mm_struct *mm,
                     struct vm_area_struct *vma, unsigned long address,
                     pte_t *pte, pmd_t *pmd, unsigned int flags)
{
        pte_t entry;
        spinlock_t *ptl;

        /*
         * some architectures can have larger ptes than wordsize,
         * e.g.ppc44x-defconfig has CONFIG_PTE_64BIT=y and CONFIG_32BIT=y,

         * so READ_ONCE or ACCESS_ONCE cannot guarantee atomic accesses.
         * The code below just needs a consistent view for the ifs and
         * we later double check anyway with the ptl lock held. So here
         * a barrier will do.
         */
        entry = *pte;
        barrier();
        if (!pte_present(entry)) {
                if (pte_none(entry)) {
                        if (vma->vm_ops)
                                return do_fault(mm, vma, address, pte, pmd,
                                                flags, entry);

                        return do_anonymous_page(mm, vma, address, pte, pmd,
                                        flags);
                }
                return do_swap_page(mm, vma, address,
                                        pte, pmd, flags, entry);
        }

        if (pte_protnone(entry))
                return do_numa_page(mm, vma, address, entry, pte, pmd);

        ptl = pte_lockptr(mm, pmd);
        spin_lock(ptl);
        if (unlikely(!pte_same(*pte, entry)))
                goto unlock;
        if (flags & FAULT_FLAG_WRITE) {
                if (!pte_write(entry))
                        return do_wp_page(mm, vma, address,
                                        pte, pmd, ptl, entry);
                entry = pte_mkdirty(entry);
        }
        entry = pte_mkyoung(entry);
        if (ptep_set_access_flags(vma, address, pte, entry, flags & FAULT_FLAG_WRITE)) {
                update_mmu_cache(vma, address, pte);
        } else {
                /*
                 * This is needed only for protection faults but the arch code
                 * is not yet telling us if this is a protection fault or not.
                 * This still avoids useless tlb flushes for .text page faults
                 * with threads.
                 */
                if (flags & FAULT_FLAG_WRITE)
                        flush_tlb_fix_spurious_fault(vma, address);
        }
unlock:
        pte_unmap_unlock(pte, ptl);
        return 0;
}

```

buffered_rmqueue code flow:
![Alt text](/pic/buffered_rmqueue.png)

```c
Buddy system.

Main structure.

On NUMA machines, each NUMA node would have a pg_data_t to describe it's memory layout
typedef struct pglist_data {
        struct zone node_zones[MAX_NR_ZONES];
        struct zonelist node_zonelists[MAX_ZONELISTS];
        int nr_zones;
...
}pg_data_t;
```

***
Each Node has only two zonelist:
1. one for all zones with memory
2. one just containing zones from the node the zonelist belongs to.
***
```c
This zone list contains a maximum of MAXNODES*MAX_NR_ZONES zones
(include/linux/mmzone.h)
struct zonelist {
        struct zonelist_cache *zlcache_ptr;                  // NULL or &zlcache
        struct zoneref _zonerefs[MAX_ZONES_PER_ZONELIST + 1];
#ifdef CONFIG_NUMA
        struct zonelist_cache zlcache;                       // optional ...
#endif
};

(include/linux/mmzone.h)
struct zone {
...
  /* free areas of different sizes */
  struct free_area free_area[MAX_ORDER];
...
}

(include/linux/mmzone.h)
struct free_area {
          struct list_head        free_list[MIGRATE_TYPES];
          unsigned long           nr_free;
};


The organization of "ZONE" in memory:
 _________
|____0____|           free_list  ---------> free_list
|____1____|          ______1_________       _______2________  ... nr_free
|____2____| <=====> |_1_|_2_|_3_|_4_|<===>|_1_|_2_|_3_|_4_|  ...
    . . ^^                                 ^^
    . . ||=================================||             
    . .
|MAX_ORDER|
```
The relationship between buddy system and pages.                              
The First page organized by free_area->list_head, because the pages are  consecutive
![Alt text](/pic/buddy_page)

The memory point and zonelist
![Alt text](/pic/backlist)
***
__alloc_pages_nodemask() can be divided into two parts:
1. Fast path
2. Slow path

Fast path: Go through the Water Mark search the suitable zone in zonelist.    
Slow path: Do two things.
					 1. Swap the inactive pages into swap areas.
					 2. Kill the thread which keep more memory.

**NOTE:** Regarding the "migratetype" in free_list, in my opinion, all of the free_list[] pointed every different size memory block start address of page, so, "migratetype" means whether this memory block can be migrated or not.

***
```c
/*
 * This is the 'heart' of the zoned buddy allocator.
 */
struct page *
__alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order,
                        struct zonelist *zonelist, nodemask_t *nodemask)
{
        struct zoneref *preferred_zoneref;
        struct page *page = NULL;
        unsigned int cpuset_mems_cookie;
        int alloc_flags = ALLOC_WMARK_LOW|ALLOC_CPUSET|ALLOC_FAIR;
        gfp_t alloc_mask; /* The gfp_t that was actually used for allocation */
        struct alloc_context ac = {
                .high_zoneidx = gfp_zone(gfp_mask),
                .nodemask = nodemask,
                .migratetype = gfpflags_to_migratetype(gfp_mask),
        };

        gfp_mask &= gfp_allowed_mask;

        lockdep_trace_alloc(gfp_mask);

        might_sleep_if(gfp_mask & __GFP_WAIT);

        if (should_fail_alloc_page(gfp_mask, order))
                return NULL;

        /*
         * Check the zones suitable for the gfp_mask contain at least one
         * valid zone. It's possible to have an empty zonelist as a result
         * of __GFP_THISNODE and a memoryless node
         */
        if (unlikely(!zonelist->_zonerefs->zone))
                return NULL;

        if (IS_ENABLED(CONFIG_CMA) && ac.migratetype == MIGRATE_MOVABLE)
                alloc_flags |= ALLOC_CMA;

retry_cpuset:
        cpuset_mems_cookie = read_mems_allowed_begin();

        /* We set it here, as __alloc_pages_slowpath might have changed it */
        ac.zonelist = zonelist;
        /* The preferred zone is used for statistics later */
        preferred_zoneref = first_zones_zonelist(ac.zonelist, ac.high_zoneidx,
                                ac.nodemask ? : &cpuset_current_mems_allowed,
                                &ac.preferred_zone);
        if (!ac.preferred_zone)
                goto out;
        ac.classzone_idx = zonelist_zone_idx(preferred_zoneref);

        /* First allocation attempt */
        alloc_mask = gfp_mask|__GFP_HARDWALL;
        page = get_page_from_freelist(alloc_mask, order, alloc_flags, &ac);
        if (unlikely(!page)) {
                /*
                 * Runtime PM, block IO and its error handling path
                 * can deadlock because I/O on the device might not
                 * complete.
                 */
                alloc_mask = memalloc_noio_flags(gfp_mask);

                page = __alloc_pages_slowpath(alloc_mask, order, &ac);
        }

        if (kmemcheck_enabled && page)
                kmemcheck_pagealloc_alloc(page, order, gfp_mask);

        trace_mm_page_alloc(page, order, alloc_mask, ac.migratetype);

out:
        /*
         * When updating a task's mems_allowed, it is possible to race with
         * parallel threads in such a way that an allocation can fail while
         * the mask is being updated. If a page allocation is about to fail,
         * check if the cpuset changed during allocation and if so, retry.
         */
        if (unlikely(!page && read_mems_allowed_retry(cpuset_mems_cookie)))
                goto retry_cpuset;

        return page;
}
EXPORT_SYMBOL(__alloc_pages_nodemask);


```

![Alt text](/pic/overview.jpeg)

***
This is the Fast path for allocating Pages
***

```c
/*
 * get_page_from_freelist goes through the zonelist trying to allocate
 * a page.
 */
static struct page *
get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags,
                                                const struct alloc_context *ac)
{
        struct zonelist *zonelist = ac->zonelist;
        struct zoneref *z;
        struct page *page = NULL;
        struct zone *zone;
        nodemask_t *allowednodes = NULL;/* zonelist_cache approximation */
        int zlc_active = 0;             /* set if using zonelist_cache */
        int did_zlc_setup = 0;          /* just call zlc_setup() one time */
        bool consider_zone_dirty = (alloc_flags & ALLOC_WMARK_LOW) &&
                                (gfp_mask & __GFP_WRITE);
        int nr_fair_skipped = 0;
        bool zonelist_rescan;

zonelist_scan:
        zonelist_rescan = false;

        /*
         * Scan zonelist, looking for a zone with enough free.
         * See also __cpuset_node_allowed() comment in kernel/cpuset.c.
         */
        for_each_zone_zonelist_nodemask(zone, z, zonelist, ac->high_zoneidx,
                                                                ac->nodemask) {
                unsigned long mark;

                if (IS_ENABLED(CONFIG_NUMA) && zlc_active &&
                        !zlc_zone_worth_trying(zonelist, z, allowednodes))
                                continue;
                if (cpusets_enabled() &&
                        (alloc_flags & ALLOC_CPUSET) &&
                        !cpuset_zone_allowed(zone, gfp_mask))
                                continue;
                /*
                 * Distribute pages in proportion to the individual
                 * zone size to ensure fair page aging.  The zone a
                 * page was allocated in should have no effect on the
                 * time the page has in memory before being reclaimed.
                 */
                if (alloc_flags & ALLOC_FAIR) {
                        if (!zone_local(ac->preferred_zone, zone))
                                break;
                        if (test_bit(ZONE_FAIR_DEPLETED, &zone->flags)) {
                                nr_fair_skipped++;
                                continue;
                        }
                }
                /*
                 * When allocating a page cache page for writing, we
                 * want to get it from a zone that is within its dirty
                 * limit, such that no single zone holds more than its
                 * proportional share of globally allowed dirty pages.
                 * The dirty limits take into account the zone's
                 * lowmem reserves and high watermark so that kswapd
                 * should be able to balance it without having to
                 * write pages from its LRU list.
                 *
                 * This may look like it could increase pressure on
                 * lower zones by failing allocations in higher zones
                 * before they are full.  But the pages that do spill
                 * over are limited as the lower zones are protected
                 * by this very same mechanism.  It should not become
                 * a practical burden to them.
                 *
                 * XXX: For now, allow allocations to potentially
                 * exceed the per-zone dirty limit in the slowpath
                 * (ALLOC_WMARK_LOW unset) before going into reclaim,
                 * which is important when on a NUMA setup the allowed
                 * zones are together not big enough to reach the
                 * global limit.  The proper fix for these situations
                 * will require awareness of zones in the
                 * dirty-throttling and the flusher threads.
                 */
                if (consider_zone_dirty && !zone_dirty_ok(zone))
                        continue;

                mark = zone->watermark[alloc_flags & ALLOC_WMARK_MASK];
                if (!zone_watermark_ok(zone, order, mark,
                                       ac->classzone_idx, alloc_flags)) {
                        int ret;

                        /* Checked here to keep the fast path fast */
                        BUILD_BUG_ON(ALLOC_NO_WATERMARKS < NR_WMARK);
                        if (alloc_flags & ALLOC_NO_WATERMARKS)
                                goto try_this_zone;

                        if (IS_ENABLED(CONFIG_NUMA) &&
                                        !did_zlc_setup && nr_online_nodes > 1) {
                                /*
                                 * we do zlc_setup if there are multiple nodes
                                 * and before considering the first zone allowed
                                 * by the cpuset.
                                 */
                                allowednodes = zlc_setup(zonelist, alloc_flags);
                                zlc_active = 1;
                                did_zlc_setup = 1;
                        }

                        if (zone_reclaim_mode == 0 ||
                            !zone_allows_reclaim(ac->preferred_zone, zone))
                                goto this_zone_full;

                        /*
                         * As we may have just activated ZLC, check if the first
                         * eligible zone has failed zone_reclaim recently.
                         */
                        if (IS_ENABLED(CONFIG_NUMA) && zlc_active &&
                                !zlc_zone_worth_trying(zonelist, z, allowednodes))
                                continue;

                        ret = zone_reclaim(zone, gfp_mask, order);
                        switch (ret) {
                        case ZONE_RECLAIM_NOSCAN:
                                /* did not scan */
                                continue;
                        case ZONE_RECLAIM_FULL:
                                /* scanned but unreclaimable */
                                continue;
                        default:
                                /* did we reclaim enough */
                                if (zone_watermark_ok(zone, order, mark,
                                                ac->classzone_idx, alloc_flags))
                                        goto try_this_zone;

                                /*
                                 * Failed to reclaim enough to meet watermark.
                                 * Only mark the zone full if checking the min
                                 * watermark or if we failed to reclaim just
                                 * 1<<order pages or else the page allocator
                                 * fastpath will prematurely mark zones full
                                 * when the watermark is between the low and
                                 * min watermarks.
                                 */
                                if (((alloc_flags & ALLOC_WMARK_MASK) == ALLOC_WMARK_MIN) ||
                                    ret == ZONE_RECLAIM_SOME)
                                        goto this_zone_full;

                                continue;
                        }
                }

try_this_zone:
                page = buffered_rmqueue(ac->preferred_zone, zone, order,
                                                gfp_mask, ac->migratetype);
                if (page) {
                        if (prep_new_page(page, order, gfp_mask, alloc_flags))
                                goto try_this_zone;
                        return page;
                }
this_zone_full:
                if (IS_ENABLED(CONFIG_NUMA) && zlc_active)
                        zlc_mark_zone_full(zonelist, z);
        }

        /*
         * The first pass makes sure allocations are spread fairly within the
         * local node.  However, the local node might have free pages left
         * after the fairness batches are exhausted, and remote zones haven't
         * even been considered yet.  Try once more without fairness, and
         * include remote zones now, before entering the slowpath and waking
         * kswapd: prefer spilling to a remote zone over swapping locally.
         */
        if (alloc_flags & ALLOC_FAIR) {
                alloc_flags &= ~ALLOC_FAIR;
                if (nr_fair_skipped) {
                        zonelist_rescan = true;
                        reset_alloc_batches(ac->preferred_zone);
                }
                if (nr_online_nodes > 1)
                        zonelist_rescan = true;
        }

        if (unlikely(IS_ENABLED(CONFIG_NUMA) && zlc_active)) {
                /* Disable zlc cache for second zonelist scan */
                zlc_active = 0;
                zonelist_rescan = true;
        }

        if (zonelist_rescan)
                goto zonelist_scan;

        return NULL;
}

```

***
Why per-CPU define HOT and COLD page, it is because the different scenario.
1. HOT Page: Used for CPU in cache.
2. COLD Page: Used for Outside device, like DMA.
***
```c
buffered_rmqueue()
{
	...
	if (likely(order == 0))
	{
		...
		/* Obtain a specified number of elements from the buddy allocator */
		pcp->count += rmqueue_bulk()
		...
		/* Attempt to get the one page from per-CPU cache for enhancement */
		if (cold)
			page = list_entry(list->prev, struct page, lru);
		else
			page = list_entry(list->next, struct page, lru);
		}
		else
		{
			...
			/* Do the hard work of removing an element from the buddy allocator */
			page = __rmqueue(zone, order, migratetype);
			...
		}
		...
}

```

```c
/*              
 * Do the hard work of removing an element from the buddy allocator.
 * Call me with the zone->lock already held.
 */
static struct page *__rmqueue(struct zone *zone, unsigned int order,
                                                int migratetype)
{
        struct page *page;

retry_reserve:
        page = __rmqueue_smallest(zone, order, migratetype);

        if (unlikely(!page) && migratetype != MIGRATE_RESERVE) {
                if (migratetype == MIGRATE_MOVABLE)
                        page = __rmqueue_cma_fallback(zone, order);

                if (!page)
                        page = __rmqueue_fallback(zone, order, migratetype);

                /*
                 * Use MIGRATE_RESERVE rather than fail an allocation. goto
                 * is used because __rmqueue_smallest is an inline function
                 * and we want just one call site
                 */
                if (!page) {
                        migratetype = MIGRATE_RESERVE;
                        goto retry_reserve;
                }
        }

        trace_mm_page_alloc_zone_locked(page, order, migratetype);
        return page;
}
```
***
__rmqueue_smallest() scan all of the zone list, until fine the suitable continue pages
***

```c
/*
 * Go through the free lists for the given migratetype and remove
 * the smallest available page from the freelists
 */
static inline
struct page *__rmqueue_smallest(struct zone *zone, unsigned int order,
                                                int migratetype)
{               
        unsigned int current_order;
        struct free_area *area;
        struct page *page;

        /* Find a page of the appropriate size in the preferred list */
        for (current_order = order; current_order < MAX_ORDER; ++current_order) {
                /* Get the free_area */
                area = &(zone->free_area[current_order]);

                /* judge whether it is empty */
                if (list_empty(&area->free_list[migratetype]))
                        continue;

                /* Get the page from the area->free_list */
                page = list_entry(area->free_list[migratetype].next,
                                                        struct page, lru);

                /* Delte the page from the area->free_list */
                list_del(&page->lru);

                /* Set the PG_buddy bit Flag, this page will be not in the buddy system. then set page->private = 0 */
                rmv_page_order(page);

                /* Decrease the total number of the nr_free in this order area[oder]->free_list */
                area->nr_free--;

                /* If the pages is bigger than requirement, it means we need to allocate the memory from the high order areas, so, at this point, we need to split it as smaller block based on buddy system mechanism */
                expand(zone, page, order, current_order, area, migratetype);
                set_freepage_migratetype(page, migratetype);
                return page;
        }

        return NULL;
}

```
***
The steps of expand:
![Alt text](/pic/steps_expand.png)
***
```c
/*              
 * The order of subdivision here is critical for the IO subsystem.
 * Please do not alter this order without good reasons and regression
 * testing. Specifically, as large blocks of memory are subdivided,
 * the order in which smaller blocks are delivered depends on the order
 * they're subdivided in this function. This is the primary factor
 * influencing the order in which pages are delivered to the IO
 * subsystem according to empirical testing, and this is also justified
 * by considering the behavior of a buddy system containing a single
 * large block of memory acted on by a series of small allocations.
 * This behavior is a critical factor in sglist merging's success.
 *                                              
 * -- nyc
 */
static inline void expand(struct zone *zone, struct page *page,
        int low, int high, struct free_area *area,
        int migratetype)
{       
        unsigned long size = 1 << high;

      /*
       * area -- : move the free_list pointer
       * high -- : It is the order that kernel selected
       * size >>= 1 : It means order menus 1
       * low : It means the order wanted to allocate
       * &page[size] : It means the start address of the low order (high --)
       */
        while (high > low) {
                area--;
                high--;
                size >>= 1;
                VM_BUG_ON_PAGE(bad_range(zone, &page[size]), &page[size]);

                if (IS_ENABLED(CONFIG_DEBUG_PAGEALLOC) &&
                        debug_guardpage_enabled() &&
                        high < debug_guardpage_minorder()) {
                        /*
                         * Mark as guard pages (or page), that will allow to
                         * merge back to allocator when buddy will be freed.
                         * Corresponding page table entries will not be touched,
                         * pages will stay not present in virtual address space
                         */
                        set_page_guard(zone, &page[size], high, migratetype);
                        continue;
                }

                /* Put the &page[size].lru link the low order together */
                list_add(&page[size].lru, &area->free_list[migratetype]);
                area->nr_free++;

                /* Set struct page->private and set PG_buddy bit Flag */
                set_page_order(&page[size], high);
        }
}

```

***
If __rmqueue_smallest() can't find suitable pages, then call __rmqueue_fallbak() to try to search other
***
```c
/* Remove an element from the buddy allocator from the fallback list */
static inline struct page *
__rmqueue_fallback(struct zone *zone, unsigned int order, int start_migratetype)
{                                               
        struct free_area *area;
        unsigned int current_order;
        struct page *page;
        int fallback_mt;
        bool can_steal;

        /* Find the largest possible block of pages in the other list */
        for (current_order = MAX_ORDER-1;
                                current_order >= order && current_order <= MAX_ORDER-1;
                                --current_order) {
                area = &(zone->free_area[current_order]);
                fallback_mt = find_suitable_fallback(area, current_order,
                                start_migratetype, false, &can_steal);
                if (fallback_mt == -1)
                        continue;

                page = list_entry(area->free_list[fallback_mt].next,
                                                struct page, lru);
                if (can_steal)
                        steal_suitable_fallback(zone, page, start_migratetype);

                /* Remove the page from the freelists */
                area->nr_free--;
                list_del(&page->lru);
                rmv_page_order(page);

                expand(zone, page, order, current_order, area,
                                        start_migratetype);
                /*
                 * The freepage_migratetype may differ from pageblock's
                 * migratetype depending on the decisions in
                 * try_to_steal_freepages(). This is OK as long as it
                 * does not differ for MIGRATE_CMA pageblocks. For CMA
                 * we need to make sure unallocated pages flushed from
                 * pcp lists are returned to the correct freelist.
                 */
                set_freepage_migratetype(page, start_migratetype);

                trace_mm_page_alloc_extfrag(page, order, current_order,
                        start_migratetype, fallback_mt);

                return page;
        }

        return NULL;
}

```

Put the Huge Page into hugepage_freelists[]
***

Compound page: Only First page named Head, all of the others named Tail

PG_compound is the Flag for Compound page.

***

![Alt text](/pic/compound_page.png)


```c
void put_page(struct page *page)
{
        if (unlikely(PageCompound(page)))
                put_compound_page(page);
        else if (put_page_testzero(page))
                __put_single_page(page);
}



```



```c
wrsadmin@pek-cdong-u145:~$ cat /proc/buddyinfo
Node 0, zone      DMA      1      1      0      0      2      1      1      0     1      1      3
Node 0, zone    DMA32   1476   1253    842    590    663    827    557    361   265      3    487
Node 0, zone   Normal   6323   4159   2998   2031   2181   3306   2247   1442  1047     11   1720

nr_free                    0      1      2      3      4      5      6      7     8      9     10
                       <---------------------------------------------------------------------------->
left                      1476  1253   842    590    663    827    557    361    265     3     487

```

## Hugetlb MMap

```c
1.Hugetlb has no read/write operation, all of the action via MMAP completed.
2.MMAP does not allocate physical page, only set the offset in VMA.

MMAP:

const struct file_operations hugetlbfs_file_operations = {
        .read_iter              = hugetlbfs_read_iter,
        .mmap                   = hugetlbfs_file_mmap,
        .fsync                  = noop_fsync,
        .get_unmapped_area      = hugetlb_get_unmapped_area,
        .llseek         = default_llseek,
};

hugetlbfs_file_mmap
  hugetlb_reserve_pages

```

## Hugetlb Init

```c
Huge page allocation:

hugetlb_init
 hugetlb_init_hstates
  hugetlb_hstate_alloc_pages
   alloc_fresh_huge_page
    alloc_fresh_huge_page_node
     alloc_pages_exact_node
      __alloc_pages
       __alloc_pages_nodemask ====> allocate the huge page
    prep_new_huge_page
     put_page ===============> put the huge page into hugepage_freelists[]


Huge FS Init:

```
