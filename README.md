## TF-MoDISco

[![DOI](https://zenodo.org/badge/62352963.svg)](https://zenodo.org/badge/latestdoi/62352963)

**NOTE: we are still refining the multi-task version of TF-MoDISco. If you encounter difficulties running TF-MoDISco with multiple tasks, our recommendation is to run it on one task at a time.**

Installation:
At the time of writing, the latest version on pypi is version 0.5.6.2 and can be installed using `pip install modisco`. To install from this source code, clone the repo and then run `pip install --editable /path/to/cloned/repo`.

A technical note describing version 0.5.1.1 is available at [https://arxiv.org/abs/1811.00416](https://arxiv.org/abs/1811.00416).
Video of talk at NIPS MLCB: https://www.youtube.com/watch?v=fXPGVJg956E

Please see the following example notebooks:
- [TF MoDISco TAL GATA](examples/simulated_TAL_GATA_deeplearning/TF%20MoDISco%20TAL%20GATA.ipynb): a self-contained example notebook that uses pre-computed importance scores (generated by a neural network) as input. Scores were generated using deeplift as illustated in [this notebook](examples/simulated_TAL_GATA_deeplearning/Generate%20Importance%20Scores.ipynb)
- [TF MoDISco Nanog](examples/H1ESC_Nanog_gkmsvm/TF%20MoDISco%20Nanog.ipynb): a self-contained example notebook that uses pre-computed importance scores and an empirically-generated null distribution (generated by a gkm-SVM) as input. Scores were generated using gkmexplain as illustated in [this notebook](examples/H1ESC_Nanog_gkmsvm/Nanog_GkmExplain_Generate_Data.ipynb). This notebook also illustrates how to use a MEME-based initialization to boost the performance of TF-MoDISco.

TF-MoDISco has been used in the following papers:
- [Deep learning at base-resolution reveals motif syntax of the cis-regulatory code](https://www.biorxiv.org/content/10.1101/737981v1) (Avsec & Weilert et al.)
- [GkmExplain: fast and accurate interpretation of nonlinear gapped k-mer SVMs](https://academic.oup.com/bioinformatics/article/35/14/i173/5529147) (Shrikumar, Prakash & Kundaje)

Full paper on the way.

## Loading a saved TF-MoDISco HDF5 File

In the example notebooks, you will notice that the output of TF-MoDISco is saved as an HDF5 file. Below is documentation on how to load this output and what the different attributes mean. If you catch something that appears to be out-of-date or doesn't make sense, please file a github issue to let me know.

The easiest way to load the hdf5 file is to create a TfModiscoResults object via the function `modisco.tfmodisco_workflow.workflow.TfModiscoResults.from_hdf5(...)`. The use of this function is demonstrated in cell 10 of [this notebook](https://github.com/kundajelab/tfmodisco/blob/948d62c5143f4e05469f63610e7c9cf2033f0f76/examples/simulated_TAL_GATA_deeplearning/With%20Hit%20Scoring%20TF%20MoDISco%20TAL%20GATA.ipynb); the only catch is that it requires the data for all the importance score tracks to be provided via a `TrackSet` object (the `TrackSet` object is needed to recreate the seqlets from the data stored in the hdf5 file). Below I have documented the important attributes of the `TfModiscoResults` class and the key subclasses. If for whatever reason you are specifically interested in the hdf5 format, let me know and I can detail where all these attributes wind up in the hdf5 file. Alternatively you can inspect the `save_hdf5` functions of the relevant classes to see how the attributes are stored.

`tfmodisco_workflow.workflow.TfModiscoResults`:
- `.task_names`: list of the task names that `TfModiscoWorkflow` object was called with
- `.multitask_seqlet_creation_results`: instance of `core.MultitaskSeqletCreationResults`; stores all the information about the seqlets that were identified across all tasks during the seqlet identification step. See below.
- `.metaclustering_results`: instance of `metaclusterers.MetaclusteringResults`, which stores details on the metaclusters obtained from doing metaclustering on the seqlets. See below.
- `.metacluster_idx_to_submetacluster_results`: dictionary that maps the metacluster number to an instance of `tfmodisco_workflow.workflow.SubMetaclusterResults`. `SubMetaclusterResults` stores the results of applying clustering to the seqlets within a metacluster, including the motifs found for the metacluster. See below.

`tfmodisco_workflow.workflow.SubMetaclusterResults`:
- `.metacluster_size`: the number of seqlets in this metacluster
- `.activity_pattern`: the activity pattern of this metacluster. The activity pattern of a metacluster is a vector of length=number-of-tasks, and the entries in the vector are -1, 0 or 1 for each task. The activity pattern indicates how the seqlets in that metacluster contribute to the different tasks.
- `.seqlets`: the seqlets that fell within this metacluster
- `.seqlets_to_patterns_result`: an instance of `tfmodisco_workflow.seqlets_to_patterns.SeqletsToPatternsResults`; this stores information on the motifs ("patterns") identified within the metacluster. See below.

`tfmodisco_workflow.seqlets_to_patterns.SeqletsToPatternsResults`
- `.success`: whether or not the motif discovery for this metacluster terminated successfully
- `.patterns`: a list of instances of core.AggregatedSeqlet, which represent the motifs. See below.
- `.total_time_taken`: the total time taken for performing motif discovery for this metacluster.

`core.AggregatedSeqlet`: (this is the class used to represent motifs)
- `.seqlets` returns a list of seqlets for this motif. seqlets are instances of the `core.Seqlet` class. See below
- `[track_name].fwd`: returns the forward strand version of track_name; this is the average value over all seqlets in the motif. (Note: in case my notation is unclear, I mean that you can use the dictionary lookup syntax, i.e. do `motif[track_name].fwd` to get the data). `track_name` is a string.
- `[track_name].rev`: returns the reverse complement of track_name; this is the average value over all seqlets in the motif

`core.Seqlet`:
- `.coor`: returns an instance of core.SeqletCoordinates; see below
- `[track_name].fwd`: returns the forward strand version of track_name
- `[track_name].rev`: returns the reverse complement version of track_name

`core.SeqletCoordinates`:
- `.example_idx`: the index of the example from which this seqlet originated. This index corresponds to the data that was provided in the call to TfModiscoWorkflow.
- `.start`: the location of the start of the seqlet within the example
- `.end`: the location of the end of the seqlet within the example
- `.is_revcomp`: whether the seqlet is on the forward or the reverse strand

`core.MultitaskSeqletCreationResults`
- `.multitask_seqlet_creator`: instance of `core.MultitaskSeqletCreator`; stores the information needed to create the seqlets given new data
- `.final_seqlets`: the final list of seqlets produced across tasks
- `.task_name_to_coord_producer_results`: mapping from the task name to an instance of coordproducers.CoordProducerResults, which stores information on the seqlet coordinates identified for that particular task, as well as the thresholding cutoffs used.

`metaclusterers.MetaclusteringResults`
- `.metacluster_indices`: a vector where `metacluster_indices[i]` returns the metacluster number for the seqlet at index i. You should find that the ordering matches the ordering of `TfModiscoResults.multitask_seqlet_creation_results.final_seqlets`.
- `.metaclusterer`: an instance of `metaclusterers.AbstractMetaclusterer`, which can be used to assign metaclusters to seqlets obtained on new data.
- `.attribute_vectors`: mostly for debugging purposes; this would be the attributes that were extracted to the seqlets and supplied to the `Metaclusterer` for metaclustering; think of them as the seqlet features that were used for metaclustering. Again, the order of the `attribute_vectors` should match the ordering of `TfModiscoResults.multitask_seqlet_creation_results.final_seqlets`.
- `.metacluster_idx_to_activity_pattern`: mapping from the metacluster to the pattern of activity across tasks. The activity pattern of a metacluster is a vector of length=number-of-tasks, and the entries in the vector are -1, 0 or 1 for each task. It indicates how the seqlets in that metacluster contribute to the different tasks.
