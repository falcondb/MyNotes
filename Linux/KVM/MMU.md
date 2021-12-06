## MMU in KMV

### [Documentation/virtual/kvm/mmu](https://www.kernel.org/doc/Documentation/virtual/kvm/mmu.txt)
- Events
  - Guest generated events:
    - writes to control registers (especially cr3)
    - invlpg/invlpga instruction execution
    - access to missing or protected translations
  - Host generated events:
    - changes in the gpa->hpa translation (either through gpa->hva changes or
      through hva->hpa changes)
    - memory pressure (the shrinker)

- Shadow pages

### [The x86 kvm shadow mmu](https://www.kernel.org/doc/Documentation/virtual/kvm/mmu.txt)
- Translation
  - When the number of required translations matches the hardware, the mmu operates in direct mode; otherwise it operates in shadow mode.

- Memory
  - These hvas may be backed using any method available to the host: anonymous memory, file backed memory, and device memory.  Memory might be paged by the host at any time.

- Events
  - Guest generated events
    - writes to control registers (especially cr3)
    - invlpg/invlpga instruction execution
    - access to missing or protected translations
  - Host generated events
    - changes in the gpa->hpa translation
    - memory pressure


- Shadow pages (`struct kvm_mmu_page`)

  - A shadow page contains 512 sptes, which can be either leaf or nonleaf sptes.  A shadow page may contain a mix of leaf and nonleaf sptes.

  - A leaf spte corresponds to either one or two translations encoded into one paging structure entry.  These are always the lowest level of the translation stack, with optional higher level translations left to NPT/EPT.

  - The fields of `struct kvm_mmu_page`
    - role.level: 1=4k sptes, 2=2M sptes, 3=1G sptes
    - role.direct: If set, leaf sptes reachable from this page are for a linear range. The linear range starts at (gfn << PAGE_SHIFT) and its size is determined by role.level; If clear, this page corresponds to a guest page table denoted by the gfn
    field
    - gfn: Either the guest page table containing the translations shadowed by this page, or the base page frame for linear translations. See role.direct.
    - spt:
      - A pageful of 64-bit sptes containing the translations for this page. Accessed by both kvm and hardware.
      - *The page pointed to by `spt` will have its `page->private` pointing back at the shadow page structure.*
      - The spt array forms a DAG structure with the shadow page as a node, and guest pages as leaves.
    - gfns: An array of 512 guest frame numbers, one for each present pte.  Used to perform a reverse map from a pte to a gfn.  
    - parent_ptes: The reverse mapping for the pte/ptes pointing at this page's spt

- Reverse map
  - The mmu maintains a reverse mapping whereby all ptes mapping a page can be reached given its gfn.

- Synchronized and unsynchronized pages
  - The guest uses two events to synchronize its tlb and page tables: tlb flushes and page invalidations (`invlpg`)
  - A tlb flush means that we need to synchronize all sptes reachable from the guest's cr3. This is expensive, so we keep all guest page tables write protected, and synchronize sptes to gptes when a gpte is written.
  - A special case is when a guest page table is reachable from the current guest cr3.  In this case, the guest is obliged to issue an invlpg instruction before using the translation.  We take advantage of that by removing write protection from the guest page, and allowing the guest to modify it freely. We synchronize modified gptes when the guest invokes invlpg.

- Reaction to events
  - guest page fault, The cause of a page fault can be:
    - a true guest fault
    - access to a missing translation
    - access to a protected translation
    - access to untranslatable memory`


### [kvm: virtual x86 mmu setup](https://blog.stgolabs.net/search/label/x86)
