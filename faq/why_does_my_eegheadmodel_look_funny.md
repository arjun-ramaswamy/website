---
title: Why does my EEG headmodel look funny?
tags: [faq, freq]
---

# Why does my EEG headmodel look funny?

If you create an EEG (or MEG) headmodel from an anatomical MRI image, it may happen that the one-shoe-fits-almost-all recipe that you are following does not work that wel. For instance, using the pipeline sketched in the [EEG BEM headmodel](/tutorial/headmodel_eeg_bem) tutorial, it could be that the meshes you end up with contain artifacts. For instance, there may be undesired 'horns' attached to the head or brain surface, or there might be holes. The below code shows two situations that might happen. Sometimes, things can be relatively easily solved, by tweaking judiciously the tweakable parameters in the pipeline used, but sometimes there's no way around going in by hand and to manually adjust/fix the faulty segmented images. This FAQ only sketches 2 situations that can be solved relatively easily. The first example, uses the single subject template MRI from FieldTrip, which has quite a prominent aliasing artifact at the top of the image. Using a segmentation approach with the default settings will result in a 'strange' head surface. This can be alleviated with tweaking some segmentation parameters. The second example introduces an manipulated MRI image, that has an inhomogeneous intensity. This might occur for instance if the participant was wearing an EEG cap with electrodes while the image was acquired. Satisfactory results can be obtained if the intensity bias in the image is corrected prior to segmenting. For this, one can used the function **[ft_volumebiascorrect](https://github.com/fieldtrip/fieldtrip/blob/release/plotting/ft_volumebiascorrect.m)**.

    % load an anatomical mri
    [ftver, ftpath] = ft_version;
    mri = ft_read_mri(fullfile(ftpath, 'template/anatomy', 'single_subj_T1_1mm.nii'));
    mri.coordsys = 'acpc';

The image on disk has an ugly aliasing artifact at the top, causing the standard settings to create an ugly segmentation. increasing the threshold for the scalp segmentation, and switching off the smoothing is sufficient to fix it in this case. But let's first show that it leads to a segmentation problem, and a funky scalp surface.

    cfg          = [];
    cfg.output   = {'brain','skull','scalp'};
    segmentedmri0 = ft_volumesegment(cfg, mri);

    cfg             = [];
    cfg.tissue      = {'brain','skull','scalp'};
    cfg.numvertices = [3000 2000 1000];
    bnd0            = ft_prepare_mesh(cfg, segmentedmri0);

    figure;
    ft_plot_mesh(bnd0(3), 'facecolor',[0.4 0.4 0.4]);
    view([0 0]);

{% include image src="/assets/img/faq/why_does_my_eegheadmodel_look_funny/bnd0.png" width="350" %}

_Figure 1. Aliasing artifact at the top leads to horns_

Adjusting the settings for the segmentation is helpful in this case, but will not always work.
    
    cfg          = [];
    cfg.output   = {'brain','skull','scalp'};
    cfg.scalpthreshold = 0.2;
    cfg.scalpsmooth  = 'no';
    segmentedmri = ft_volumesegment(cfg, mri);

    cfg             = [];
    cfg.tissue      = {'brain','skull','scalp'};
    cfg.numvertices = [3000 2000 1000];
    bnd             = ft_prepare_mesh(cfg, segmentedmri);

    figure;
    ft_plot_mesh(bnd(3), 'facecolor',[0.4 0.4 0.4]);
    view([0 0]);

{% include image src="/assets/img/faq/why_does_my_eegheadmodel_look_funny/bnd1.png" width="350" %}

_Figure 1. Adjustment of segmentation parameters gets rid of the aliasing artifact_

the chunk of code below is intended to create a mask for the relevant
anatomical data, so that the aliasing artifact can be zero'ed out, so
that we don't need to account for that anymore
    scalpmask = double(segmentedmri.scalp);

fill the bottom
    scalpmask(50:130,50:130,1) = 1;
    pw_dir = pwd;
    cd(fullfile(ftpath, 'private'));
    scalpmask = volumefillholes(scalpmask);
    cd(pw_dir);

    mri2 = mri;
    mri2.anatomy(~scalpmask) = 0; % avoid the strange aliasing effect

change the homogeneity of the image
    krn  = gausswin(181,2)*gausswin(201,2)';
    K    = repmat(krn, [1 1 180]);
    for k = 1:180
      K(:,:,k) = K(:,:,k).*(k./150);
    end
    blob = ones(mri2.dim);
    blob(91+[-90:90], 101+[-100:100], 2:end) = max(1-K,0);

    mri2.anatomy = mri2.anatomy.*blob;
    ft_sourceplot([], mri2);

    cfg          = [];
    cfg.output   = {'brain','skull','scalp'};
    segmentedmri2 = ft_volumesegment(cfg, mri2);

    cfg             = [];
    cfg.tissue      = {'brain','skull','scalp'};
    cfg.numvertices = [3000 2000 1000];
    bnd2            = ft_prepare_mesh(cfg, segmentedmri2);
the above already throws a warning that the segmentation is not
star-shaped

    figure;
    ft_plot_mesh(bnd2(3), 'facecolor',[0.4 0.4 0.4]);
    view([90 0]);

estimate the inhomogeneity and remove this bias
    mri3 = ft_volumebiascorrect([], mri2);

    cfg          = [];
    cfg.output   = {'brain','skull','scalp'};
    segmentedmri3 = ft_volumesegment(cfg, mri3);

    cfg             = [];
    cfg.tissue      = {'brain','skull','scalp'};
    cfg.numvertices = [3000 2000 1000];
    bnd3            = ft_prepare_mesh(cfg, segmentedmri3);

    figure;
    ft_plot_mesh(bnd3(3), 'facecolor',[0.4 0.4 0.4]);
    view([90 0]);