# BowyerWatson
Implementation of the Bowyer-Watson algorithm (with modified Lloyd relaxation) for finding Delaunay triangulations of pointsets on the unit sphere.

************ EXECUTION ************

The existing code generates $pointcount random points on the unit sphere (seeding srand with the seed $seed) and finds a triangulation using the Bowyer-Watson algorithm; this is performed ($relaxations + 1) times with an application of modified Lloyd relaxation in between executions. The output is a python2 file which utilizes mpl_toolkits.mplot3d and matplotlib.pyplot to present an interactive 3D rendering of the resulting mesh. Direct compilation of the code therefore results in an executable designed for testing the code's efficacy and viewing the results; (simple and straightforward) modifications are required to output the results in a different format, or to take the initial pointset from external input rather than randomly generating it--see the section on MODIFICATION below, as well as main.cpp and the comments in bowyer_watson.h for usage.

The help menu can be accessed by passing "--help" to the executable as its only parameter.

Otherwise, the expected parameters--all optional--and their defaults are as follows:
  1) $pointcount = 1000 : A positive integer specifying the number of points to generate.
  2) $relaxations = 5: A nonnegative integer specifying the number of times to perform modified Lloyd relaxation.
  3) $filename = "bwoutput.py": A string designating the desired filename for the python2 script output. ".py" will be appended if absent.
  4) $seed = 666: An integer to seed the random number generator (srand).

************ COMPILATION ************

You must compile with support for C++11. External requirements are limited to the Eigen header-only library and Intel Threading Building Blocks; the Eigen folder must be within an include directory, and the TBB library must be linked against. Suggested compilation parameters:

g++ -ltbb -I"/wherever/eigen/is/located" main.cpp bowyer_watson.cpp

************ MODIFICATION ************

See main.cpp and the comments in bowyer_watson.h for usage. In short:

(1) Initialize an std::vector<std::pair<Eigen::Vector3f, unsigned int> > to contain the points to be used in the triangulation. These can be generated by BowyerWatson::generate_points(int how_many_points_to_generate, std::vector<...> &vector_to_them_in). One will observe that the vector contains not mere points, but points paired with unsigned integers: the integer associated with a point should be between 0 and 7 inclusive, and can be arbitrary--but for maximal efficiency, the integer chosen should designate the quadrant of Euclidean 3-space containing the point according to the following legend (where u/d mean up/down, n/s mean north/south, and e/w mean east/west, and where the positive z axis is up, the positive y axis is north, and the positive x axis is east): { 0: uen, 1: unw, 2: use, 3: uws, 4: dne, 5: dwn, 6: des, 7: dsw }.

(2) Pass the point vector to BowyerWatson::perform(BowyerWatson::create_initial_triangles(your_point_vector)), obtaining a pointer to a BowyerWatson::Triangle as its return value. This triangle is the head of a linked list of all the triangles produced by the triangulation. Alternatively to create_initial_triangles, you may use BowyerWatson::create_cardinal_triangles; this will add 6 points to your pointset, one at each intersection of the unit sphere with each of the x, y, and z axes.

(3) Call BowyerWatson::relax_points(Triangle *triangle_pointer, vector<...> &point_vector), passing the triangle and a reference to your point vector (or to a new one). The point vector will be cleared and filled with the new points resulting from the relaxation procedure. Then call BowyerWatson::perform(BowyerWatson::create_initial_triangles(point_vector)) on the point vector again to obtain a new triangulation. Do not use create_cardinal_triangles here, or you will end up inserting points at the intersections of the unit sphere with the coordinate axes repeatedly. Repeat this as often as you wish, depending on how much relaxation you wish to apply to the pointset.

(4) Call bw.get_mesh(Triangle *triangle_pointer, bool true_to_call_delete_on_all_the_triangles) to obtain a BowyerWatson::Mesh object representing the triangle mesh you have created in a more useful form. The mesh consists of an std::vector<float> storing the vertices (the Nth point's x,y,z coordinates are vertices[3*n], vertices[3*N+1], and vertices[3*N+2], respectively) and an std::vector<unsigned int> storing the indices of the vertices belonging to particular triangles (the Mth triangles vertices are the triangles[3*M]th, triangles[3*M+1]th, and triangles[3*M+2]th points, respectively, in counterclockwise order).

NOTE: If you are seeking tilings of the sphere by Voronoi polygons, they can readily be constructed from the Delaunay triangulations produced by this program, being dual to them.

************ POTENTIAL ISSUES ************

(1) There may be issues with pointsets containing multiple copies of the same point; at the very least, this will result in a mesh containing degenerate triangles.

(2) create_initial_triangles (as opposed to create_cardinal_triangles) potentially has issues if your pointset is limited to a small area of the sphere or a very small number of points, because it constructs the initial triangulation using the 6 points which lie farthest up, down, north, south, east, and west, and does not currently verify that the resulting polyhedron is a valid and non-degenerate triangulation. You will certainly have issues if your pointset consists of fewer than 6 points.

(3) Modified Lloyd relaxation may not produce nice results if your triangulation includes a triangle so large that it wraps around a significant portion of the sphere, because area and centroids are calculated in Euclidean 3-space; the assumption is that the algorithm will be used on sufficiently large and well-distributed pointsets that this does not become an issue.

************ NOTES ON DESIGN ************

(1) By "modified Lloyd relaxation" is meant the following: the area and centroid of each triangle is calculated in 3D Euclidean space, and the centroid is projected onto the surface of the unit sphere (hence the "modified": properly, this should be done in a spherical geometry, but provided the mesh is fairly dense this method is more efficient and suffices to obtain nearly the same results). Each point is then replaced by the centroid of the triangles which possess it as a vertex--that is, the average of the centroids of these triangles weighted by their respective areas, projected onto the unit sphere.

(2) For purposes of efficiency in development, the algorithm does not maintain any structure of triangles for use in determining which triangle contains a point when that point is inserted. Instead, all points are generated at the outset, and the algorithm maintains within each triangle an array of all those points which are contained within it; when triangles are destroyed and replaced with others in the course of the algorithm, points contained within the destroyed triangles are copied into the appropriate new triangle. This benefits greatly from parallelization, though an efficient data structure for determining containment  might be superior.

(3) Intel's Threading Building Blocks library is utilized in two places: first, it is utilized in sorting points contained within triangles being removed into the appropriate newly created triangles, and second, it is utilized in actually copying the points into the new triangles. Special care is taken to minimize cache invalidations during these processes; the cache size is not queried from the system but is assumed to be 64, though this can be changed within the BowyerWatson class. The question of when datasets are sufficiently large to benefit from using Intel's parallel_for rather than processing them serially has been addressed by tweaking several parameters, which are marked by comments in the code, according to what values empirically gave the best results on my own machine.

(4) Multithreading could also be used in point generation and mesh construction, as well as modified Lloyd relaxation, but at present it is not (in fact, the mesh generation algorithm could be greatly improved). Within the Bowyer-Watson algorithm, the process of discovering which triangles should be removed could be parallelized, and if multiple non-overlapping sets of removable triangles could be discovered simultaneously they could be processed in paraallel; this, however, would require substantial modifications to the code and would likely require checks too complex to justify the benefit, and so it has not been done.

************ LICENSE AND ATTRIBUTION ************

Created by Joshua Brinsfield (sapphous at gmail.com) in 2017.
Released under the MIT license (see license file).
I was unable to find a simple implementation of Delaunay triangulation on the sphere, so I made this. Have a blast, if this is useful to you.
