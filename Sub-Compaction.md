Here we explain the sub-compaction that is used in both Leveled and Universal compactions.

### Goal
The goal of sub-compaction is to speed up a compaction job by partitioning it among multiple threads.

### When
It is employed when one of the following conditions holds:
* L0 -> Lo where o > 0
  * Why: L0->Lo cannot be run in parallel with another L0->Lo, hence partitioning is the only way to speed it up.
* Universal Compaction, except L0 -> L0.
* Manual Leveled Compaction: Li -> Lo where o > 0
  * Why: Manual compaction invoked by the user is usually not parallelized hence benefits from partitioning.

Note: sub-compaction is disabled for leveled if it is not merged with any file from the target level Lo. Refer to Compaction::ShouldFormSubcompactions for details.

### How
It currently is based on a heuristic that worked out well so far. The heuristic could be improved in many ways.
1. Select boundaries based on the natural boundary of input levels/files.
   * first and last key of L0 files 
   * first and last key of non-0, non-last levels
   * first key of each SST file of the last level
1. Use Versions::ApproximateSize to estimate the data size in each boundary.
1. Merge boundaries to eliminate empty and smaller-than-average ranges.
   * find the average size in each range
   * starting from beginning greedily merge adjacent ranges until their total size exceeds the average
