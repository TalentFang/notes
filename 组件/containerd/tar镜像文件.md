calico-node.tar
├── manifest.json              ← Docker 的顶层索引（非 OCI，是 docker 自己的格式）
├── repositories               ← 旧版 Docker 用的，新版已废弃
└── blobs/
    └── sha256/
        ├── ac1682e9...         ← Manifest（镜像清单）
        ├── 54637cb36...        ← Config（镜像配置）
        ├── 928dad07...         ← Layer 0（文件系统层）
        ├── 6ab78488...         ← Layer 1
        └── ...



ctr image import 之后会多两条记录：
	1. tag记录，名字，可读性
	2. config 哈希，作用：1)containerd的GC判定是否有人使用image的依据 2)kubelet的cri接口用config哈希来定位镜像