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
    
