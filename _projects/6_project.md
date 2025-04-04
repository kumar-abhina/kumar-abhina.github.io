---
layout: page
title: One Shot Imitation Learning via Interaction Warping
description: A detailed exploration of one-shot imitation learning using Interaction Warping for SE(3) manipulation tasks.
tags: imitation learning, interaction warping, shape warping, robotics
img: assets/Publication/biza23oneshot.gif
importance: 6
category: work
---

# Introduction

**Figure 1: The Mug Tree task.**

In one-shot imitation learning, we are given a single demonstration of a desired manipulation behavior and we must find a policy that can reproduce the behavior in different situations. A classic example is the *Mug Tree* task, where a robot must grasp a mug and hang it on a tree by its handle. Given a single demonstration of grasping a mug and hanging it on a tree (top row of Figure 1), we want to obtain a policy that can successfully generalize across objects and poses (e.g. differently-shaped mugs and trees, as shown in the bottom row of Figure 1). This presents two key challenges:
- The demonstration must generalize to novel object instances (e.g. different mugs).
- The policy must reason in SE(3) rather than in SE(2), where the problem is much easier [1].

To be successful in SE(3) manipulation, it is generally necessary to bias the model significantly toward the object manipulation domains in question. One popular approach is to establish a correspondence between points on the surfaces of the objects in the demonstration(s) and the same points on objects seen at test time. This is typically implemented using **keypoints**—point descriptors that encode the semantic location on an object’s surface and transfer well between novel instances [2, 3, 4]. For example, points on handles from different mugs should be assigned similar descriptors to aid in matching. A key challenge, therefore, becomes how to learn semantically meaningful keypoint descriptors. Early work used hand-coded feature labels [4], while more recent methods learn category-level object descriptor models during a pre-training step using implicit object models [5] or point models [2].

*7th Conference on Robot Learning (CoRL 2023), Atlanta, USA*  
*arXiv:2306.12392v2 [cs.RO] 4 Nov 2023*

This paper proposes a different approach to the point correspondence problem based on **Coherent Point Drift (CPD)** [6], a point-cloud warping algorithm. We call this method **Interaction Warping**. Using CPD, we train a shape-completion model to register a novel in-category object instance to a canonical object model in which the task has been defined via a single demonstration. The canonical task can then be projected into scenes with novel in-category objects by registering the new objects to the canonical models. Our method offers several advantages over previous approaches [2, 3, 4]:
1. It performs better in terms of successfully executing a novel instance of a demonstrated task—both in simulation and on robotic hardware.
2. It requires an order-of-magnitude fewer object instances to train each new object category (tens rather than hundreds).
3. It is agnostic to the use of neural networks—the approach is based on CPD and PCA models (although neural networks can be incorporated).

# Related Work

We draw on prior work in shape warping [7, 8] and imitation learning via keypoints [4].  
- **Shape Warping:** Uses non-rigid point cloud registration [9] to align point clouds or meshes of objects, transferring robot skills across objects of different shapes. Our work is the first to use shape warping for relational object re-arrangement and for handling objects in arbitrary poses.
- **Keypoints:** Serve as a state abstraction that reduces objects to a set of task-specific keypoint poses. These keypoints help transfer robot actions. The novelty here is that our interaction points are automatically discovered and warped together with the object shape.

Few-shot learning of manipulation policies has been explored using keypoint-based methods [4, 10, 11], which typically rely on human-annotated keypoints. More recent approaches have learned keypoints for tool affordances [12, 13, 14] and model-based RL [15]. Related work includes the learning of 2D [16] and 3D [5, 17, 18, 19] descriptor fields that provide semantic embeddings for arbitrary points, allowing keypoints to be matched across instances. We specifically compare our method to that of Simeonov et al. [5, 17] and show that our method requires fewer demonstrations. Other related approaches involve cross-attention between point clouds [2, 20] and pose estimation for precise object insertion [21].

# Background

**Figure 2: Coherent Point Drift warping.**

**Coherent Point Drift (CPD):**  
Given two point clouds, \(X^{(i)} \in \mathbb{R}^{n \times 3}\) and \(X^{(j)} \in \mathbb{R}^{m \times 3}\), CPD finds a displacement \(W_{i \to j} \in \mathbb{R}^{n \times 3}\) that brings the points in \(X^{(i)}\) as close as possible (in an L2 sense) to the points in \(X^{(j)}\) [6]. CPD is a non-rigid registration method—each point in \(X^{(i)}\) can be translated independently. It minimizes the following cost function:

$$
J(W_{i \to j}) = -\sum_{k=1}^{m} \log \sum_{l=1}^{n} \exp\left(-\frac{1}{2\sigma^2} \|X^{(i)}_l + (W_{i \to j})_l - X^{(j)}_k\|\right) + \frac{\alpha}{2} \, \phi(W_{i \to j}),
$$

where \(\phi(W_{i \to j})\) is a regularization term that enforces coherent movement among nearby points.

**Generative Object Modeling Using CPD:**  
Assume we are given a set of point clouds \(\{X^{(1)}, \dots, X^{(K)}\}\) describing \(K\) object instances from a single category (e.g. various mug instances). Select a “canonical” object \(X^{(C)}\) (\(C \in \{1, 2, \dots, K\}\)) and define displacement matrices \(W_{C \to i} = \text{CPD}(X^{(C)}, X^{(i)})\) for \(i=1, \dots, K\). Heuristically, we choose the most representative \(X^{(C)}\) (see Appendix A.2). We then form the flattened data matrix \(\bar{W}_C = [\bar{W}_{C \to 1}, \dots, \bar{W}_{C \to K}]\) and calculate a \(d\)-dimensional PCA projection matrix \(W \in \mathbb{R}^{3n \times d}\). This allows us to approximate novel in-category objects using a latent vector \(v_{\text{novel}} \in \mathbb{R}^d\) and compute a point cloud:

$$
Y = X^{(C)} + \text{Reshape}(W v_{\text{novel}}),
$$

where Reshape converts the vector back to an \(n \times 3\) matrix.

**Shape Completion From Partial Point Clouds:**  
To approximate a complete point cloud for an object seen partially [8], we solve:

$$
L(Y) = D(Y, X(\text{partial})),
$$

via gradient descent on \(v\). Here, \(D(\cdot,\cdot)\) is typically the one-sided Chamfer distance:

$$
D\left(X^{(i)}, X^{(j)}\right) = \frac{1}{m} \sum_{k=1}^{m} \min_{l \in \{1, \dots, n\}} \|X^{(i)}_l - X^{(j)}_k\|^2.
$$

Note that \(X^{(i)} \in \mathbb{R}^{n \times 3}\) and \(X^{(j)} \in \mathbb{R}^{m \times 3}\) may have different numbers of points.

# Interaction Warping

This section describes **Interaction Warping (IW)**, our proposed imitation method (see Figure 3). First, we train a set of category-level generative object models as described above. Then, given a single demonstration of a desired manipulation activity, we detect objects using off-the-shelf models. For each object matching a pre-trained model, we fit the model to obtain its pose and completed shape (Sections 4.1 and 4.2). Next, we identify interaction points on pairs of objects and correspond these points with those in the canonical object models. Finally, we reproduce the demonstration in a new scene by projecting the demonstrated interaction points onto completed object instances (Section 4.3).

## 4.1 Joint Shape and Pose Inference

To manipulate objects in SE(3), we jointly infer the pose and shape of an object represented by a point cloud \(X(\text{partial})\). We warp and transform the point cloud \(Y \in \mathbb{R}^{n \times 3}\) to minimize:

$$
L(Y) = D(Y, X(\text{partial})) + \beta \max_k \|Y_k\|^2,
$$

which is similar to Equation 3 but with an additional regularizer to keep the object’s size minimal (preventing oversized predicted meshes). We parameterize \(Y\) as a warped, scaled, rotated, and translated canonical point cloud:

$$
Y = \left[(X^{(C)} + \text{Reshape}(W v)) \,\Big|\, \{z\}\right] \odot s \, R^T + t.
$$

Here, \(X^{(C)}\) is a canonical point cloud, \(v \in \mathbb{R}^d\) parameterizes a warped shape, \(s \in \mathbb{R}^3\) represents scale, \(R \in SO(3)\) is a rotation matrix, and \(t \in \mathbb{R}^3\) represents translation. We optimize \(L\) with respect to \(v\), \(s\), and \(t\) using the Adam optimizer [36]. \(R\) is parameterized using an arbitrary matrix \(\hat{R} \in \mathbb{R}^{2 \times 3}\) followed by Gram-Schmidt orthogonalization (Algorithm 5) to yield a valid rotation matrix. This parameterization enables stable learning of rotations [37, 38]. We run the optimization with many random restarts (see Appendix A.4).

## 4.2 From Point Clouds to Meshes

While we infer object shape and pose by warping point clouds, collision checking and motion planning require meshes. We recover a warped mesh \(M\) by first warping the vertices of the canonical object. Since our model only warps points in \(X^{(C)}\) (Section 3), we include both the original mesh vertices and additional randomly sampled points on the canonical surface to ensure balanced warping. The first \(V\) points of \(X^{(C)}\) (which remain in order during warping) become the warped mesh vertices. These vertices, combined with the canonical faces, form the warped mesh \(M\).

## 4.3 Transferring Robot Actions via Interaction Points

**Figure 4:**  
- (a) Contacts between a gripper and a bowl extracted from a demonstration.  
- (b) Nearby points between a mug and a tree extracted from a demonstration of hanging the mug on the tree.  
- (c) A virtual point (red) representing the branch of the tree intersecting the mug’s handle. The red point is anchored to the mug using \(k\) nearest neighbors (four shown in green).

Consider a warped mug point cloud \(Y\) (from Equation 6). By tracking a point \(Y_i\) on the mug’s handle, we can align handles across mugs of different shapes and sizes. We call such points **interaction points**.

### Grasp Interaction Points

We define grasp interaction points as pairs of contact points between the gripper and the object at grasp. Let \(Y^{(A)}\) and \(M^{(A)}\) denote the point cloud and mesh of the grasped object (obtained as in Sections 4.1 and 4.2), and let \(M^{(G)}\) be the gripper mesh with grasp pose \(T_G\). Using pybullet collision checking, we obtain \(P\) pairs of contact points \(\left(p^{(A)}_j, p^{(G)}_j\right)\) for \(j=1,\dots,P\). Since our model warps points in \(Y^{(A)}\), we identify indices \(I_G = \{i_1, \dots, i_P\}\) such that \(Y^{(A)}_{i_j}\) is the nearest neighbor of \(p^{(A)}_j\).

### Transferring Grasps

For a new object with point cloud \(Y^{(A')}\) (via Equation 6), we compute the new grasp by finding the transformation \(T^*_G\) that best aligns the pairs \(\left(Y^{(A')}_{i_j}, p^{(G)}_j\right)\) for \(j = 1, \dots, P\). Shape warping preserves the order of points, ensuring a consistent correspondence between \(Y^{(A)}\) and \(Y^{(A')}\). The predicted grasp \(T^*_G\) minimizes the pairwise distances using Horn et al.’s method [39] (see Figure 5a).

### Placement Interaction Points

For placement tasks (e.g. placing a mug on a mug-tree), interaction points are defined as pairs of nearby points between the two objects, even if they are not in direct contact. Let \(Y^{(A)}\) and \(Y^{(B)}\) be the inferred point clouds from a demonstration captured before the gripper opens. We select pairs of nearby points with an L2 distance less than \(\delta\):

$$
\{(p^{(A)}, p^{(B)}) : \|p^{(A)} - p^{(B)}\| < \delta\}.
$$

Due to the large number of potential pairs, we sample a representative subset using farthest point sampling [40] and record the indices of \(p^{(B)}_j\) in \(Y^{(B)}\) as \(I_P = \{i_1, i_2, \dots, i_P\}\).

Furthermore, we add \(p^{(B)}_j\) as virtual points into \(Y^{(A)}\) (illustrated in Figures 4b and 4c) to ensure proper alignment when there is no natural contact point. For each \(p^{(B)}_j\), we find \(L\) nearest neighbors \((n_{j,1}, \dots, n_{j,L})\) in \(Y^{(A)}\) and anchor a virtual point \(q^{(A)}_j\) as follows:

$$
q^{(A)}_j = \frac{1}{L} \sum_{k=1}^L \left( Y^{(A)}_{n_{j,k}} + \left( p^{(B)}_j - Y^{(A)}_{n_{j,k}} \right) \Delta_{j,k} \right) = p^{(B)}_j.
$$

To transfer the placement, we save the neighbor indices \(n_{j,k}\) and displacements \(\Delta_{j,k}\).

For new objects with point clouds \(Y^{(A')}\) and \(Y^{(B')}\), we compute the warped virtual points:

$$
q^{(A')}_j = \frac{1}{L} \sum_{k=1}^L \left( Y^{(A')}_{n_{j,k}} + \Delta_{j,k} \right).
$$

We then form point pairs \(\left(q^{(A')}_j, Y^{(B')}_{i_j}\right)\) for \(j=1,\dots,P\) and determine the optimal transformation \(T^*_P\) for placing object \(A'\) onto object \(B'\) (see Figure 5b).

# Experiments

We evaluate both the perception and imitation learning capabilities of Interaction Warping.

In **Section 5.1**, we perform three object re-arrangement tasks with previously unseen objects, both in simulation and on a physical robot. In **Section 5.2**, we demonstrate that our system can propose grasps in a cluttered kitchen setting from a single RGB-D image.

We use ShapeNet [41] for per-category (mug, bowl, bottle, and box) object pre-training (required by our method and all baselines). Synthetic mug-tree meshes from [17] are also used. Our method (IW) requires only 10 training examples per class, whereas all baselines use 200 examples. All training meshes are aligned in a canonical pose.

## 5.1 Object Re-arrangement

**Setup:**  
We use an open-source simulated environment with three tasks:
- Mug on a mug-tree
- Bowl on a mug
- Bottle in a container [17]

Given a segmented point cloud of the initial scene, the goal is to predict the pose of the child object relative to the parent object (e.g., the mug relative to the mug-tree). A successful action places the object on a rack or in a container so that it does not fall, while avoiding collisions. The simulation does not test grasp prediction. These tasks are demonstrated with objects unseen during pre-training. In Section 4.3, we described how IW uses a single demonstration; when multiple demonstrations are available, IW selects the most informative one using training prediction error (see Appendix A.5).

# Conclusion

In this work, we introduced **Interaction Warping**, a novel method for one-shot imitation learning that leverages point cloud registration techniques such as CPD and PCA-based generative models. Our approach significantly reduces the number of required demonstrations and generalizes well across novel object instances in challenging SE(3) manipulation tasks.

---

*This paper was presented at the 7th Conference on Robot Learning (CoRL 2023) in Atlanta, USA.*
