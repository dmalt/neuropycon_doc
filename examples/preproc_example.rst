.. _preproc_example:

Preprocessing Workflow
======================

The following script implements a preprocessing pipeline to perform **ICA** on MEG data in **ds** format 
(see :ref:`preproc_meeg` Section). 

Before to run the script, the :ref:`params` should be be downloaded (see download
link below). The main **parameters** to set for the preprocessing pipeline are

* ``main_path`` : main path of the pipeline (*mandatory*)
* ``data_type`` : 'fif' or 'ds'
* ``l_freq`` : low cut-off frequency in Hz
* ``h_freq`` : high cut-off frequency in Hz
* ``down_sfreq`` : sampling frequency at which the data are downsampled
* ``variance`` : float between 0 and 1: the ICA components will be selected by the cumulative percentage of explained variance
* ``ECG_ch_name`` : the name of ECG channels
* ``EoG_ch_name`` : the name of EoG channels

Furthermore, the following **inputnode** should be specified:

* ``raw_file`` : path to raw MEG data in **fif** format (*mandatory*)
* ``subject_id`` : subject id

.. seealso:: see :py:func:`create_pipeline_preproc_meeg <ephypype.pipelines.preproc_meeg.create_pipeline_preproc_meeg>` for a list of all possible inputs

.. code:: python

  import nipype.pipeline.engine as pe
  import nipype.interfaces.io as nio

  from nipype.interfaces.utility import IdentityInterface

  from ephypype.pipelines.preproc_meeg import create_pipeline_preproc_meeg

  from params_ica import main_path, data_path, subject_ids, sessions
  from params_ica import preproc_pipeline_name
  from params_ica import data_type, down_sfreq, l_freq, h_freq
  from params_ica import variance, ECG_ch_name, EoG_ch_name


  def create_infosource():
      """Create node which passes input filenames to DataGrabber"""

      infosource = pe.Node(interface=IdentityInterface(fields=['subject_id',
							      'sess_index']),
			  name="infosource")

      infosource.iterables = [('subject_id', subject_ids),
			      ('sess_index', sessions)]

      return infosource


  # it could be ds or fif file. Set data_type in the params_ica.py file
  def create_datasource():
      """"Create node to grab data"""
      
      datasource = pe.Node(interface=nio.DataGrabber(infields=['subject_id',
							      'sess_index'],
						    outfields=['raw_file']),
			  name='datasource')

      datasource.inputs.base_directory = data_path
      datasource.inputs.template = '*%s/%s/meg/%s*rest*.ds'
	
      datasource.inputs.template_args = dict(raw_file=[['subject_id',
							'sess_index',
							'subject_id']])

      datasource.inputs.sort_filelist = True

      return datasource


  def create_workflow_preproc():
      """Create nodes and connect them into a workflow"""
      
      main_workflow = pe.Workflow(name=preproc_pipeline_name)
      main_workflow.base_dir = main_path

      # Info source
      infosource = create_infosource()

      # Data source
      datasource = create_datasource()

      main_workflow.connect(infosource, 'subject_id', datasource, 'subject_id')
      main_workflow.connect(infosource, 'sess_index', datasource, 'sess_index')

      preproc_workflow = create_pipeline_preproc_meeg(main_path,
						      l_freq=l_freq,
						      h_freq=h_freq,
						      down_sfreq=down_sfreq,
						      variance=variance,
						      ECG_ch_name=ECG_ch_name,
						      EoG_ch_name=EoG_ch_name,
						      data_type=data_type)

      main_workflow.connect(infosource, 'subject_id',
			    preproc_workflow, 'inputnode.subject_id')

      main_workflow.connect(datasource, 'raw_file',
			    preproc_workflow, 'inputnode.raw_file')

      return main_workflow


  if __name__ == '__main__':

      # run pipeline:
      main_workflow = create_workflow_preproc()

      main_workflow.write_graph(graph2use='colored')  # colored
      main_workflow.config['execution'] = {'remove_unnecessary_outputs': 'false'}

      # Run workflow locally on 2 CPUs
      main_workflow.run(plugin='MultiProc', plugin_args={'n_procs': 8})


The outputs of the pipeline will be saved in the pipeline directory specified by ``main_workflow.base_dir = main_path``:

* **ica solution** file saved in $main_path/$preproc_pipeline_name/preproc_meeg/_sess_index_#_subject_id_#/ica/$raw_file_ica_solution.fif
* **cleaned raw** data file saved in $main_path/$preproc_pipeline_name/preproc_meeg/_sess_index_#_subject_id_#/ica/$raw_file_ica.fif
* **report html** file saved in $main_path/$preproc_pipeline_name/preproc_meeg/_sess_index_#_subject_id_#/ica/$raw_file-report.html

To visually inspect the ICA components, the report file can be visualized by a jupyter notebook where it is 
possible include or exclude the ICA components inspecting their topographies and time series.

.. warning:: The new cleaned raw data file should be saved in the **subject data directory**.

**Download** Parameters file: :download:`params_ica.py <../../examples/params_ica.py>`

**Download** Python source code: :download:`run_preprocess_pipeline.py <../../examples/run_preprocess_pipeline.py>`

**Download** Jupyter notebook: :download:`ipynb_report.ipynb <../../examples/ipynb_report.ipynb>`

.. warning:: To use the jupyter notebook, the following packages should be installed
.. code-block:: bash 

    conda install ipywidgets
    conda install matplotlib
    conda install 'pyqt<5'