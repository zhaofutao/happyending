# 一、KDTree简介
        KDTree(k-dimensional树的简称)，是一种分隔k维数据空间的数据结构，主要应用在多维空间关键数据的搜索（如范围搜索和最近邻搜索）。
![blockchain](/resource/images/kdtree.png "KDTree")  
        
        上面的树就是一颗KDTree，形似二叉搜索树，其实KDTree就是二叉搜索树的变种，这里K = 3。  
        首先来看下树的组织原则，将每一个元祖按0排序（第一项序号为0，第二项序号为1，第三项序号为2），在树的第n层，第n%K项就被用粗体显示，而被这些粗体显示的树就是作为二叉搜索树的key值。
        比如，根节点的左子树中每一个节点的第一项均小于根节点的第一项，右子树的根节点中第一项均大于根节点的第一项，子树依次类推。
        对于这样的一棵树，对其进行搜索节点会非常容易，给定一个元组，首先和根节点比较第一项，小于往左，大于往右，第二项比较第二项，依次类推。

![blockchain](/resource/images/kdtree-1.png)  
![blockchain](/resource/images/kdtree-2.png)