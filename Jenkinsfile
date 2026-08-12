<![CDATA[
// Source: https://github.com/jenkinsci/docker-plugin (MIT License)
// Test case: Jenkins plugin build with multi-platform testing
/*
 See the documentation for more options:
 https://github.com/jenkins-infra/pipeline-library/
*/
buildPlugin(
  useContainerAgent: false, // Set to `true` if Docker not required for containerized tests
  configurations: [
    [platform: 'linux', jdk: 21],
    [platform: 'windows', jdk: 17],
])
    ]]>