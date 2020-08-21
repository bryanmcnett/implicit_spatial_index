# implicit_spatial_index

Often in computer graphics we need to determine when one thing might touch other things in the world. It is possible to iterate over all of the things in the world, and test for intersection against each thing in turn. When most things do not move often, it is common instead to build a spatial index, which makes it possible to trivially reject large groups of things without considering each individually.

Here I present a spatial index that is an implicit data structure: no data is stored, other than the objects themselves. This is achieved by "sorting" the objects.

Imagine an array of 20 rectangles, which we will think of as 5 contiguous sets of 4 contiguous rectangles. We sort the array by increasing minimum-X, and pull elements 0, 5, 10, and 15 out into a set of 4: 0,5,10,15. This leaves four sets of four, with the initial indices 1,2,3,4, 6,7,8,9, 11,12,13,14, 16,17,18,19. Each of these four sets remains sorted by increasing value of minimum-X, and all 4 sets together remain sorted by increasing value of minimum-X.

We wish to see if one rectangle intersects any of the 20 rectangles above. We can do this for rectangles [0,5,10,15] using four AABB intersection tests. But, while doing so we also take note of for which of the four comparisons query.maximum_x >= database.minimum_x is false. If this is false for rectangle 0, then logically speaking it must also be false for 1,2,3,4, because their minimum-X is never less than rectangle 0's. For each of the four rectangles [0,5,10,15] we just AABB tested, a false query.maximum_x >= database.minimum_x can trivially reject another set of 4 rectangles.

After doing 4 AABB tests and looking at each of the four query.maximum_x >= database.minimum_x comparisons, one of five results is possible:

1. 0,1,2,3 are false: the query rectangle can not intersect any of the 20 rectangles
2. 1,2,3 are false: the query rectangle may intersect 1 of the 4 we AABB tested, and may intersect 1 of the remaining 4 sets of 4.
3. 2,3 are false: the query rectangle may intersect 2 of the 4 we AABB tested, and may intersect 2 of the remaining 4 sets of 4.
4. 3 is false: the query rectangle may intersect the 3 of the 4 we AABB tested, and may intersect 3 of the remaining 4 sets of 4.
5. none are false: the query rectangle may intersect any of the 20 rectangles

Then, for each of the sets we have not tested, but which could possibly intersect, we can recursively apply the same steps. In the first round we compare minimum-X, and in the second round we compare maximum-X. Then minimum-Y, and then maximum-Y, and then minimum-X again, and so on. 

Imagine that the original set of 20 rectangles is 84 rectangles: the first 4 tell us which of the following 4 sets of 20 could possibly intersect, and then each set of 20 begins with 4 that tell us which of the following 4 sets of 4 could possibly intersect. This can be extended to 340, 1364, etc.

4 is convenient because it is a common SIMD width, which enables fast parallel AABB tests. But this number can be 8, or 16, or 32. The larger this number, the more finely-grained we can trivially reject: 4 gives us 0/4, 1/4, 2/4, 3/4, and 4/4. 32 gives us from 0/32 up to 32/32, in steps of 1/32. However, this gives us fewer rounds in which to cycle through the axes.
