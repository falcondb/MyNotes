[Introduction to RCU homepage](http://www.rdrop.com/users/paulmck/RCU/)
[Resizable, Scalable, Concurrent Hash Tables via Relativistic Programming](https://www.usenix.org/legacy/event/atc11/tech/final_files/Triplett.pdf)

[Review Checklist for RCU Patches](https://www.kernel.org/doc/Documentation/RCU/checklist.txt)
- Is RCU being applied to a read-mostly situation?  If the data structure is updated more than about 10% of the time, then you should strongly consider some other approach
