#!/usr/bin/env groovy

// Generated from snippet generator 'properties; set job properties'
properties([buildDiscarder(logRotator(
    artifactDaysToKeepStr: '',
    artifactNumToKeepStr: '',
    daysToKeepStr: '',
    numToKeepStr: '10')),
    disableConcurrentBuilds(),
    // parameters([booleanParam( name: 'push_image_to_docker_hub', defaultValue: false, description: 'Push tensile image to rocm docker-hub' )]),
    [$class: 'CopyArtifactPermissionProperty', projectNames: '*']
   ])

////////////////////////////////////////////////////////////////////////
// -- AUXILLARY HELPER FUNCTIONS
// import hudson.FilePath;
import java.nio.file.Path;

////////////////////////////////////////////////////////////////////////
// Return build number of upstream job
@NonCPS
int get_upstream_build_num( )
{
    def upstream_cause = currentBuild.rawBuild.getCause( hudson.model.Cause$UpstreamCause )
    if( upstream_cause == null)
      return 0

    return upstream_cause.getUpstreamBuild()
}

////////////////////////////////////////////////////////////////////////
// Return project name of upstream job
@NonCPS
String get_upstream_build_project( )
{
    def upstream_cause = currentBuild.rawBuild.getCause( hudson.model.Cause$UpstreamCause )
    if( upstream_cause == null)
      return null

    return upstream_cause.getUpstreamProject()
}

////////////////////////////////////////////////////////////////////////
// Calculate the relative path between two sub-directories from a common root
@NonCPS
String g_relativize( String root_string, String rel_source, String rel_build )
{
  Path root_path = new File( root_string ).toPath( )
  Path path_src = root_path.resolve( rel_source )
  Path path_build = root_path.resolve( rel_build )

  String rel_path = path_build.relativize( path_src ).toString( )

  // If rel_path is empty,
  if( rel_path?.trim( ) )
    return rel_path
  else
    return "."
}

////////////////////////////////////////////////////////////////////////
// Construct the relative path of the build directory
void build_directory_rel( project_paths paths, compiler_data hcc_args )
{
  // if( hcc_args.build_config.equalsIgnoreCase( 'release' ) )
  // {
  //   paths.project_build_prefix = paths.build_prefix + '/' + paths.project_name + '/release';
  // }
  // else
  // {
  //   paths.project_build_prefix = paths.build_prefix + '/' + paths.project_name + '/debug';
  // }

  // Currently, for this python based project, the build directory has to match the source directory
  paths.project_build_prefix = paths.project_src_prefix;
}

////////////////////////////////////////////////////////////////////////
// Lots of images are created above; no apparent way to delete images:tags with docker global variable
def docker_clean_images( String org, String image_name )
{
  // Check if any images exist first grepping for image names
  int docker_images = sh( script: "docker images | grep \"${org}/${image_name}\"", returnStatus: true )

  // The script returns a 0 for success (images were found )
  if( docker_images == 0 )
  {
    // run bash script to clean images:tags after successful pushing
    sh "docker images | grep \"${org}/${image_name}\" | awk '{print \$1 \":\" \$2}' | xargs docker rmi"
  }
}

////////////////////////////////////////////////////////////////////////
// -- BUILD RELATED FUNCTIONS

////////////////////////////////////////////////////////////////////////
// Checkout source code, source dependencies and update version number numbers
// Returns a relative path to the directory where the source exists in the workspace
void checkout_and_version( project_paths paths )
{
  paths.project_src_prefix = paths.src_prefix + '/' + paths.project_name

  dir( paths.project_src_prefix )
  {
    sh """#!/usr/bin/env bash
      set -x
      ls -la
    """

    // checkout tensile
    checkout([
      $class: 'GitSCM',
      branches: scm.branches,
      doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
      extensions: scm.extensions + [[$class: 'CleanCheckout']],
      userRemoteConfigs: scm.userRemoteConfigs
    ])

    sh """#!/usr/bin/env bash
      set -x
      ls -la
    """
  }
}

////////////////////////////////////////////////////////////////////////
// This creates the docker image that we use to build the project in
// The docker images contains all dependencies, including OS platform, to build
def docker_build_image( docker_data docker_args, project_paths paths )
{
  String build_image_name = "build-tensile-hip-artifactory"
  def build_image = null

  dir( paths.project_src_prefix )
  {
    def user_uid = sh( script: 'id -u', returnStdout: true ).trim()

    // Docker 17.05 introduced the ability to use ARG values in FROM statements
    // Docker inspect failing on FROM statements with ARG https://issues.jenkins-ci.org/browse/JENKINS-44836
    // build_image = docker.build( "${paths.project_name}/${build_image_name}:latest", "--pull -f docker/${build_docker_file} --build-arg user_uid=${user_uid} --build-arg base_image=${from_image} ." )

    // JENKINS-44836 workaround by using a bash script instead of docker.build()
    sh "docker build -t ${paths.project_name}/${build_image_name}:latest -f docker/${docker_args.build_docker_file} ${docker_args.docker_build_args} --build-arg user_uid=${user_uid} --build-arg base_image=${docker_args.from_image} ."
    build_image = docker.image( "${paths.project_name}/${build_image_name}:latest" )
  }

  return build_image
}

////////////////////////////////////////////////////////////////////////
// This encapsulates the cmake configure, build and package commands
// Leverages docker containers to encapsulate the build in a fixed environment
def docker_build_inside_image( def build_image, compiler_data compiler_args, docker_data docker_args, project_paths paths )
{
  // Construct a relative path from build directory to src directory; used to invoke cmake
  String rel_path_to_src = g_relativize( pwd( ), paths.project_src_prefix, paths.project_build_prefix )

  build_image.inside( docker_args.docker_run_args )
  {
    // PYTHONPATH is used for setup.py install --prefix install_prefix
    // 02/05/2018 - PATH is to run unit tests, however setting path does not appear to work in withEnv block
    withEnv(["PATH+TENSILE=./install_prefix/bin/", "PYTHONPATH=install_prefix/lib/python2.7/site-packages/"])
    {
      // The following does it's best to not use sudo; setting install prefix to local directory
      // Running setup.py as root creates files/directories owned by root, which is a pain
      sh """#!/usr/bin/env bash
        set -x
        cd ${paths.project_build_prefix}
        mkdir -p install_prefix/lib/python2.7/site-packages
        python ${rel_path_to_src}/setup.py install --prefix install_prefix
      """

      stage( "Test ${compiler_args.compiler_name} ${compiler_args.build_config}" )
      {
        // Cap the maximum amount of testing to be a few hours; assume failure if the time limit is hit
        timeout(time: 2, unit: 'HOURS')
        {
          // Because PATH does not appear to get set with 'withEnv', manually set it in the bash script below
          sh  """#!/usr/bin/env bash
              set -x
              export PATH=\${PATH}:./install_prefix/bin/
              cd ${paths.project_build_prefix}

              # defaults
              tensile ${rel_path_to_src}/Tensile/Configs/test_hgemm_defaults.yaml hgemm_defaults
              tensile ${rel_path_to_src}/Tensile/Configs/test_sgemm_defaults.yaml sgemm_defaults
              tensile ${rel_path_to_src}/Tensile/Configs/test_dgemm_defaults.yaml dgemm_defaults

              # thorough tests
              tensile ${rel_path_to_src}/Tensile/Configs/test_hgemm.yaml hgemm
              tensile ${rel_path_to_src}/Tensile/Configs/test_sgemm.yaml sgemm

              # vectors
              tensile ${rel_path_to_src}/Tensile/Configs/test_hgemm_vectors.yaml hgemm_vectors
              tensile ${rel_path_to_src}/Tensile/Configs/test_sgemm_vectors.yaml sgemm_vectors

              # tensor contractions
              tensile ${rel_path_to_src}/Tensile/Configs/test_tensor_contraction.yaml tensor_contraction

              # assembly
              tensile ${rel_path_to_src}/Tensile/Configs/test_sgemm_asm.yaml sgemm_asm
              tensile ${rel_path_to_src}/Tensile/Configs/test_dgemm_asm.yaml dgemm_asm
          """

          // TODO re-enable when jenkins supports opencl
          //sh "tensile --runtime-language=OCL ../../Tensile/Configs/test_sgemm_vectors.yaml sgemm_vectors"
          //sh "tensile --runtime-language=OCL ${rel_path_to_src}/Tensile/Configs/convolution.yaml tensor_contraction"

          archiveArtifacts artifacts: "${paths.project_build_prefix}/dist/*.egg", fingerprint: true
        }
      }
    }
  }

  return void
}

// Docker related variables gathered together to reduce parameter bloat on function calls
class docker_data implements Serializable
{
  String from_image
  String build_docker_file
  String install_docker_file
  String docker_run_args
  String docker_build_args
}

// Docker related variables gathered together to reduce parameter bloat on function calls
class compiler_data implements Serializable
{
  String compiler_name
  String build_config
  String compiler_path
}

// Paths variables bundled together to reduce parameter bloat on function calls
class project_paths implements Serializable
{
  String project_name
  String src_prefix
  String project_src_prefix
  String build_prefix
  String project_build_prefix
}

////////////////////////////////////////////////////////////////////////
// -- MAIN
// Following this line is the start of MAIN of this Jenkinsfile

// Integration testing is a special path which implies testing of an upsteam build of hcc,
// but does not need testing across older builds of hcc or cuda.
// params.hip_integration_test is set in HIP build
// NOTE: hip does not currently set this bit; this is non-functioning at this time
// if( params.hip_integration_test )
// {
//   println "Enabling tensile integration testing pass"

//   node('docker && rocm')
//   {
//     hip_integration_testing( '--device=/dev/kfd', 'hip-ctu', 'Release' )
//   }

//   return
// }

// This defines a common build pipeline used by most targets
def build_pipeline( compiler_data compiler_args, docker_data docker_args, project_paths tensile_paths, def docker_inside_closure )
{
  ansiColor( 'vga' )
  {
    stage( "Build ${compiler_args.compiler_name} ${compiler_args.build_config}" )
    {
      // Checkout source code, dependencies and version files
      checkout_and_version( tensile_paths )

      // Conctruct a binary directory path based on build config
      build_directory_rel( tensile_paths, compiler_args );

      // Create/reuse a docker image that represents the tensile build environment
      def tensile_build_image = docker_build_image( docker_args, tensile_paths )

      // Print system information for the log
      tensile_build_image.inside( docker_args.docker_run_args, docker_inside_closure )

      // Build tensile inside of the build environment
      docker_build_inside_image( tensile_build_image, compiler_args, docker_args, tensile_paths )
    }
  }
}

// The following launches 3 builds in parallel: hcc-ctu, hcc-rocm and cuda
parallel hcc_ctu:
{
  node( 'docker && rocm && dkms' )
  {
    def docker_args = new docker_data(
        from_image:'compute-artifactory:5001/rocm-developer-tools/hip/master/hip-hcc-ctu-ubuntu-16.04:latest',
        build_docker_file:'dockerfile-build-hip-hcc-ctu-ubuntu-16.04',
        install_docker_file:'dockerfile-tensile-hip-hcc-ctu-ubuntu-16.04',
        docker_run_args:'--device=/dev/kfd --device=/dev/dri --group-add=video',
        docker_build_args:' --pull' )

    def compiler_args = new compiler_data(
        compiler_name:'hcc-ctu',
        build_config:'Release',
        compiler_path:'/opt/rocm/bin/hcc' )

    def tensile_paths = new project_paths(
        project_name:'tensile',
        src_prefix:'src',
        build_prefix:'src' )

    def print_version_closure = {
      sh  """
          set -x
          /opt/rocm/bin/rocm_agent_enumerator -t ALL
          /opt/rocm/bin/hcc --version
        """
    }

    build_pipeline( compiler_args, docker_args, tensile_paths, print_version_closure )
  }
},
hcc_rocm:
{
  node( 'docker && rocm && !dkms' )
  {
    def hcc_docker_args = new docker_data(
        from_image:'rocm/rocm-terminal:latest',
        build_docker_file:'dockerfile-build-rocm-terminal',
        install_docker_file:'dockerfile-tensile-rocm-terminal',
        docker_run_args:'--device=/dev/kfd --device=/dev/dri --group-add=video',
        docker_build_args:' --pull' )

    def hcc_compiler_args = new compiler_data(
        compiler_name:'hcc-rocm',
        build_config:'Release',
        compiler_path:'/opt/rocm/bin/hcc' )

    def tensile_paths = new project_paths(
        project_name:'tensile',
        src_prefix:'src',
        build_prefix:'src' )

    def print_version_closure = {
      sh  """
          set -x
          /opt/rocm/bin/rocm_agent_enumerator -t ALL
          /opt/rocm/bin/hcc --version
        """
    }

    build_pipeline( hcc_compiler_args, hcc_docker_args, tensile_paths, print_version_closure )
  }
}
"""
,
nvcc:
{
  node( 'docker && cuda' )
  {
    def hcc_docker_args = new docker_data(
        from_image:'nvidia/cuda:8.0-devel',
        build_docker_file:'dockerfile-build-nvidia-cuda-8',
        install_docker_file:'dockerfile-install-nvidia-cuda-8',
        docker_run_args:'--device=/dev/nvidiactl --device=/dev/nvidia0 --device=/dev/nvidia-uvm --device=/dev/nvidia-uvm-tools --volume-driver=nvidia-docker --volume=nvidia_driver_375.74:/usr/local/nvidia:ro',
        docker_build_args:' --pull' )

    def hcc_compiler_args = new compiler_data(
        compiler_name:'nvcc-8.0',
        build_config:'Release',
        compiler_path:'g++' )

    def tensile_paths = new project_paths(
        project_name:'tensile',
        src_prefix:'src',
        build_prefix:'src' )

    def print_version_closure = {
      sh  """
      //    set -x
      //    nvidia-smi
      //    nvcc --version
        """
    }

    build_pipeline( hcc_compiler_args, hcc_docker_args, tensile_paths, print_version_closure )
  }
}
"""
