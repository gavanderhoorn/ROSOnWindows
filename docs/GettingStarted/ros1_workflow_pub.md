```yaml
name: CD
on:
  push:
    # Sequence of patterns matched against refs/tags
    tags:
    - 'v*' # Push events to matching v*, i.e. v1.0, v20.15.10

jobs:
  build_and_test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
          os: [windows-latest]
    steps:
    - uses: actions/checkout@v2
    - name: Cleanup windows environment
      shell: bash
      run: |
        rm -rf /c/hostedtoolcache/windows/Boost/1.72.0/lib/cmake/Boost-1.72.0
        mkdir -p /c/ci
        cp $GITHUB_WORKSPACE/ci/toolchain.cmake /c/ci
    - uses: goanpeca/setup-miniconda@v1
      with:
        activate-environment: myenv
        environment-file: ci/environment.yaml
        python-version: 3.7
    - uses: ros-tooling/action-ros-ci@master
      with:
        package-name: <ros package>
        vcs-repo-file-url: ${{ github.workspace }}/ci/deps.rosinstall
        extra-cmake-args: "-G Ninja -DCMAKE_TOOLCHAIN_FILE=c:/ci/toolchain.cmake -DCMAKE_BUILD_TYPE=Release"
      env:
        COLCON_DEFAULTS_FILE: ${{ github.workspace }}/ci/defaults.yaml
        ROS_PYTHON_VERSION: 3
        CC: cl.exe
        CXX: cl.exe
    - uses: ros-tooling/action-ros-ci@master
      with:
        package-name: winml_tracker
        vcs-repo-file-url: ${{ github.workspace }}/ci/empty.rosinstall
        extra-cmake-args: "-G Ninja -DCMAKE_TOOLCHAIN_FILE=c:/ci/toolchain.cmake -DCMAKE_BUILD_TYPE=Release"
      env:
        COLCON_DEFAULTS_FILE: ${{ github.workspace }}/ci/packaging.yaml
        ROS_PYTHON_VERSION: 3
        CC: cl.exe
        CXX: cl.exe        
    - name: Build Chocolatey Package
      shell: cmd
      run: |
        cd c:\opt\install
        7z a -tzip ${{ github.workspace }}\package\tools\drop.zip *
        echo $destination = 'c:\opt\ros\melodic\x64' > ${{ github.workspace }}\package\tools\chocolateyInstall.ps1
        type $GITHUB_WORKSPACE\package\chocolateyInstall_template.ps1 >> ${{ github.workspace }}\package\tools\chocolateyInstall.ps1
        mkdir c:\opt\output
        cd ${{ github.workspace }}\package
        choco pack --trace --out c:\opt\output <ros nuspec>.nuspec 
    - name: Create Release
      id: create_release
      uses: actions/create-release@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: ${{ github.ref }}
        release_name: Release ${{ github.ref }}
        draft: false
        prerelease: false
    - name: Upload Release Asset
      id: upload-release-asset 
      uses: actions/upload-release-asset@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        upload_url: ${{ steps.create_release.outputs.upload_url }} 
        asset_path: c:/opt/output/<ros nuspec>.<ros version>.nupkg
        asset_name: <ros nuspec>.<ros version>.nupkg
        asset_content_type: application/zip 

```