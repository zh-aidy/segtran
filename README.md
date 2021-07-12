# Few-Shot Domain-Adaptation with Polymorphic Transformers

### Datasets:

The refuge datasets (the "train", "valid", "test" splits of refuge) can be downloaded from [https://refuge.grand-challenge.org/Download/](https://refuge.grand-challenge.org/Download/) (after registration). The RIM-One and Drishti-GS (not used for DA) datasets can be downloaded from [http://medimrg.webs.ull.es/research/retinal-imaging/rim-one/](http://medimrg.webs.ull.es/research/retinal-imaging/rim-one/) and [https://cvit.iiit.ac.in/projects/mip/drishti-gs/mip-dataset2/Home.php](https://cvit.iiit.ac.in/projects/mip/drishti-gs/mip-dataset2/Home.php), respectively.

The Polyp datasets (CVC612, Kvasir and CVC-300) can be downloaded from [https://github.com/DengPingFan/PraNet](https://github.com/DengPingFan/PraNet) (search for "testing data"). 

### Training and Test of Polyformer (example on "refuge"):

**Train U-Net (source):**

`python3 train2d.py --split all --maxiter 10000 --task refuge --net unet-scratch --ds train,valid,test`

**Train Polyformer (source):**

`python3 train2d.py --split all --maxiter 3000 --task refuge --net unet-scratch --ds train,valid,test --polyformer source --cp ../model/unet-scratch-refuge-train,valid,test-02062104/iter_10000.pth --sourceopt allpoly`

Suppose the command line above saves checkpoints in ../model/unet-scratch-refuge-train,valid,test-02161507.

*Arguments:*

`--polyformer`: the usage mode of polyformer. `source`: do the source mode training, i.e., train the whole polyformer layer from scratch on the source domain. `target`: fine-tune the polyformer on the target domain. `none`: do not use polyformer (i.e., use vanilla U-Net).

The reason to separate the source and target mode is to decide whether to tie the K and Q of polyformer layers. **On the source mode, K and Q are always tied (identical)**. On the target mode, they are initialized to be the same, but differently optimized (so that only optimizing K and freezing other weights becomes possible).

`--sourceopt`: what parameters to optimize in the source mode of polyformer. `allpoly`: the whole polyformer layer. `allnet`: the whole network (including U-Net).

**Train Polyformer** ($\mathcal{L}_{sup}+\mathcal{L}_{adv}+K$):

`python3 train2d.py --task refuge --ds rim --split train --samplenum 5 --maxiter 1600 --saveiter 40 --net unet-scratch --cp ../model/unet-scratch-refuge-train,valid,test-07102156/iter_500.pth --polyformer target --targetopt k --bnopt affine --adv feat --sourceds train --domweight 0.002 --bs 3 --sourcebs 2 --unsupbs 2`

*Arguments for ablations:* 

`--bnopt`: whether to fine-tune the batch norms. `None`: (default) Update BN stats but do not update affine params. `affine`: Update BN stats as well as affine params. `fixstats`: Fix BN stats and do not update affine params.

`--targetopt`: what params to optimize when the polyformer is used on the target domain. `allpoly`: the whole polyformer layer, `k,q,v`: the K,Q,V projections of transformer 1, `allnet`: the whole network (including U-Net).

`--adv`: whether train with domain adversarial loss (DAL), and use what type of DAL. `None`: (default) do not use DAL. `feat`: DAL on features. `mask`: DAL on predicted masks. 

*Other arguments:*

`--domweight`: loss weight of DAL.

`--sourceds`: the source domain dataset used for DAL (labels are not used).

`--sourcebs`: the batch size for the source domain dataloader used for DAL.

`--unsupbs`: the batch size for the target domain dataloader used for DAL. Target-domain images for DAL could be much more than the few-shot supervised images, as DAL is unsupervised and the lables of these target-domain images are not used.

**Test Polyformer:**

`python3 test2d.py --gpu 1 --ds rim --split test --samplenum 5 --bs 6 --task refuge --cpdir ../model/unet-scratch-refuge-rim-03011450 --net unet-scratch --polyformer target --nosave --iters 40-1600,40`

*Arguments:*

`--nosave`: do not save predicted masks in ../prediction. By default, they are saved under this folder.

`--cpdir`: the directory from where checkpoints are loaded and tested.

`--iters`: which iteration(s) of checkpoints to load and test. `40,80,120`: load and test iterations 40, 80, 120. `40-1600,40`: load iterations of range(40,1600+40,40), i.e., 40, 80, 120, 160, ..., 1600.

### Training and Test of Baselines:

**Train $\mathcal{L}_{sup}$**:

`python3 train2d.py --task refuge --ds rim --split train --samplenum 5 --maxiter 1000 --saveiter 40 --net unet-scratch --cp ../model/unet-scratch-refuge-train,valid,test-02062104/iter_7000.pth --polyformer none`

This fine-tune the whole U-Net model trained on the source domain, with 5-shot supervision only.

**Train RevGrad** ($\mathcal{L}_{sup} + \mathcal{L}_{adv}$):

`python3 train2d.py --task refuge --ds rim --split train --samplenum 5 --maxiter 200 --saveiter 10 --net unet-scratch --cp ../model/unet-scratch-refuge-train,valid,test-02062104/iter_7000.pth --polyformer none --adv feat --sourceds train --domweight 0.002`

**Train DA-ADV (tune whole model):**

Simply substituting `--adv feat` in the above command line with `--adv mask`, as DA-ADV is DAL on predicted masks.iu**Train DA-ADV (tune last two layers):**

`python3 train2d.py --task refuge --ds rim --split train --samplenum 5 --maxiter 200 --saveiter 10 --net unet-scratch --cp ../model/unet-scratch-refuge-train,valid,test-02062104/iter_7000.pth --polyformer none --bnopt affine --adv mask --sourceds train --domweight 0.002 --optfilter outc,up4` 

*Arguments:*

`--optfilter outc,up4`: only optimizes the last two layers ("outc" and "up4" in the U-Net source code). 

In this setting, we also optimize BN affine parameters, thus adding `--bnopt affine`.

**Train ADDA** ($\mathcal{L}_{sup} + \mathcal{L}_{adv}$):

`python3 train2d.py --task refuge --ds rim --split train --samplenum 5 --maxiter 200 --saveiter 10 --net unet-scratch --cp ../model/unet-scratch-refuge-train,valid,test-02062104/iter_7000.pth --polyformer none --adv feat --sourceds train --domweight 0.002 --adda` 

The only different argument here is `--adda`, i.e., using ADDA training instead of RevGrad (default).

**Train CellSegSSDA** ($\mathcal{L}_{sup}+\mathcal{L}_{adv}\textnormal{(mask)}+\mathcal{L}_{recon}$):

`python3 train2d.py --task refuge --ds rim --split train --samplenum 5 --maxiter 200 --saveiter 10 --net unet-scratch --cp ../model/unet-scratch-refuge-train,valid,test-02062104/iter_7000.pth --polyformer none --adv mask --sourceds train --domweight 0.001 --reconweight 0.01`

CellSegSSDA uses DAL on mask (`--adv mask`), reconstruction loss (`--reconweight 0.01`) and the few-shot supervision.

*Arguments*:

`--adv mask`: use DAL on predicted masks.

`--reconweight 0.01`: specify the weight of the reconstruction loss as 0.01 (default: `0`, i.e., disable the reconstruction loss).

**Test a non-polyformer U-Net model:**

`python3 test2d.py --task refuge --ds rim --split test --samplenum 5 --bs 6 --cpdir ../model/unet-scratch-refuge-rim-03011303 --net unet-scratch --polyformer none --nosave --iters 10-200,10`

*Arguments:*

`--split`: which part(s) of the dataset(s) to use. `all`: use the whole dataset(s). `train`: use the "train" split (only containing --samplenum=5 images). `test`: use the "test" split (containing images other than the 5 images in the "train" split). 

All the models trained using different baselines are still vanilla U-Nets. Therefore, the test command line is in the same format.