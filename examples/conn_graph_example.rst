.. _conn_graph_example:

Spectral connectivity and graph Workflow
========================================

The following script allows to perform spectral connectivity and graph analysis in **sensor space**
(see :ref:`spectral_connectivity` Section and |conmat_to_graph| pipeline).

.. |conmat_to_graph| raw:: html

   <a href="http://davidmeunier79.github.io/graphpype/conmat_to_graph.html" target="_blank">create_pipeline_conmat_to_graph_density</a>


Before to run the script, the :ref:`params` should be be downloaded (see download
link below). The main **parameters** to set for the connectivity pipeline is
            
* ``main_path`` : main path of the pipeline (*mandatory*)
* ``con_method`` : connectivity measure
* ``epoch_window_length`` : epoched data

Furthermore, the following **inputnodes** should be specified:

* ``ts_file`` : path to time series in .npy format (*mandatory*)
* ``sfreq`` : sampling frequency (*mandatory*)
* ``freq_band`` : frequency bands (*mandatory*)
* ``labels_file`` : path to the file containing a list of labels associated with nodes
* ``index`` : what to add to the name of the file
         

About the ``con_method`` parameter specifying the connectivity measure, the possible options are: 
'coh', 'imcoh', 'plv', 'pli', 'wpli', 'pli2_unbiased', 'ppc', 'cohy', 'wpli2_debiased'. 

.. note:: A clear description of the above connectivity measures can be found in the MNE python page explaining the |here| function.

.. |here| raw:: html

   <a href="http://martinos.org/mne/stable/generated/mne.connectivity.spectral_connectivity.html?highlight=spectral_connectivity#mne.connectivity.spectral_connectivity" target="_blank">spectral_connectivity</a>

.. seealso:: see :py:func:`create_pipeline_time_series_to_spectral_connectivity <ephypype.pipelines.ts_to_conmat.create_pipeline_time_series_to_spectral_connectivity>` for a list of all possible inputs

.. code:: python

  import nipype.pipeline.engine as pe
  import nipype.interfaces.io as nio

  from nipype.interfaces.utility import IdentityInterface, Function

  from ephypype.preproc import create_ts
  from ephypype.pipelines.ts_to_conmat import create_pipeline_time_series_to_spectral_connectivity

  from graphpype.pipelines.conmat_to_graph import create_pipeline_conmat_to_graph_density

  from params_congraph import main_path, data_path, subject_ids, sessions
  from params_congraph import freq_band_names, con_method
  from params_congraph import correl_analysis_name, epoch_window_length
  from params_congraph import mod, con_den

  if mod:
      from params_congraph import radatools_optim


  def get_freq_band(freq_band_name):

      from params_congraph import freq_band_names, freq_bands

      if freq_band_name in freq_band_names:
	  print freq_band_name
	  print freq_band_names.index(freq_band_name)

	  return freq_bands[freq_band_names.index(freq_band_name)]


  def create_infosource():

      from params_congraph import test

      infosource = pe.Node(interface=IdentityInterface(fields=['subject_id',
			  				       'sess_index',
							       'freq_band_name']),
			   name="infosource")


      infosource.iterables = [('subject_id', subject_ids),
			      ('sess_index', sessions),
			      ('freq_band_name', freq_band_names)]

      return infosource


  def create_datasource():

      datasource = pe.Node(interface=nio.DataGrabber(infields=['subject_id',
							       'sess_index'],
						     outfields=['raw_file']),
			   name='datasource')

      datasource.inputs.base_directory = data_path
      datasource.inputs.template = '*%s/%s/meg/%s*rest*ica.fif'
      datasource.inputs.template_args = dict(raw_file=[['subject_id',
							'sess_index',
							'subject_id']])

      datasource.inputs.sort_filelist = True

      return datasource


  def create_main_workflow_spectral_modularity():

      main_workflow = pe.Workflow(name=correl_analysis_name)
      main_workflow.base_dir = main_path

      # info source
      infosource = create_infosource()

      # data source
      datasource = create_datasource()

      main_workflow.connect(infosource, 'subject_id', datasource, 'subject_id')
      main_workflow.connect(infosource, 'sess_index', datasource, 'sess_index')

      create_ts_node = pe.Node(interface = Function(input_names=['raw_fname'], 
					    output_names=['ts_file',
							  'channel_coords_file',
							  'channel_names_file',
							  'sfreq'],
					    function=create_ts),
			name='create_ts')

      main_workflow.connect(datasource, 'raw_file',
			    create_ts_node, 'raw_fname')

      spectral_workflow = \
	  create_pipeline_time_series_to_spectral_connectivity(main_path,
							       con_method=con_method,
							       epoch_window_length=epoch_window_length)
      
      main_workflow.connect(create_ts_node, 'ts_file',
			    spectral_workflow, 'inputnode.ts_file')

      main_workflow.connect(create_ts_node, 'channel_names_file',
			    spectral_workflow, 'inputnode.labels_file')

      main_workflow.connect(infosource, ('freq_band_name', get_freq_band),
			    spectral_workflow, 'inputnode.freq_band')

      main_workflow.connect(create_ts_node, 'sfreq',
			    spectral_workflow, 'inputnode.sfreq')

      graph_den_pipe = create_pipeline_conmat_to_graph_density(main_path,
							       con_den=con_den,
							       mod=mod,
							       plot=True)

      main_workflow.connect(spectral_workflow, 'spectral.conmat_file',
			    graph_den_pipe, 'inputnode.conmat_file')

      if mod:
	  graph_den_pipe.inputs.community_rada.optim_seq = radatools_optim

	  main_workflow.connect(create_ts_node, 'channel_names_file',
				graph_den_pipe, 'inputnode.labels_file')
	  main_workflow.connect(create_ts_node, 'channel_coords_file',
				graph_den_pipe, 'inputnode.coords_file')

      return main_workflow


  if __name__ == '__main__':

      # run pipeline:
      main_workflow = create_main_workflow_spectral_modularity()

      main_workflow.write_graph(graph2use='colored')  # colored
      main_workflow.config['execution'] = {'remove_unnecessary_outputs': 'false'}
      main_workflow.run(plugin='MultiProc', plugin_args={'n_procs': 8})
      
      
**Download** Parameters file: :download:`params_congraph.py <../../examples/params_congraph.py>`

**Download** Python source code: :download:`run_spectral_modularity.py <../../examples/run_spectral_modularity.py>`
