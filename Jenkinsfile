import java.text.SimpleDateFormat
//#################################

def repoSync(fullClean, fullReset, manifestRepo, manifestRev, manifestPath){
    checkout changelog: true, poll: false, scm: [$class: 'RepoScm', currentBranch: true, \
        forceSync: true, jobs: 4, manifestBranch: manifestRev, \
        manifestFile: manifestPath, manifestRepositoryUrl: manifestRepo, \
        quiet: false, resetFirst: fullClean, resetFirst: fullReset]
} 
//#################################

node('master') {
	stage('Environment'){
		def dateFormat = new SimpleDateFormat("yyyyMMdd_HHmm")
        def date = new Date()

        env.RADXA_RELEASE_TIME = dateFormat.format(date)
        env.RADXA_RELEASE_DIR = "release"
        env.RADXA_VENDOR = "rockpi-4b"
        env.RADXA_LUNCH = "rk3399_box"
        env.PATH = "/sbin:" + env.PATH

		dir('build-environment') {
			checkout scm
		}
	}

 	def uid = sh (returnStdout: true, script: 'id -u').trim()
    def gid = sh (returnStdout: true, script: 'id -g').trim()
    def environment = docker.build('android-builder:7.x', "--build-arg USER_ID=${uid} --build-arg GROUP_ID=${gid} build-environment")

    environment.inside {
		stage('repo') {
	        repoSync(true, true, "https://github.com/radxa/rockpi4-android-tv-7.1.git", "master", "rockchip_tv_nougat_release.xml")
		}
		/*
	    stage('uboot'){
	    	sh '''#!/bin/bash
	    		#uboot
				cd u-boot
				make distclean
				make rock-pi-4b-rk3399_defconfig
				./mk-uboot.sh
				cd -
	    	'''
	    }
	    stage('kernel'){
	    	sh '''#!/bin/bash
	    		# kernel
				cd kernel
				make distclean
				for hardware in $RADXA_VENDOR;
				do
				    echo "###################kernel build $hardware###################"
				    make rockchip_defconfig
				    make rk3399-$hardware.img -j4
				    mv resource.img $hardware.img
				done
				cd -
	    	'''
	    }
		stage('android') {
			sh '''#!/bin/bash
				
				# make android
				. build/envsetup.sh
				lunch "$RADXA_LUNCH"-userdebug
				make -j8

				prebuilts/sdk/tools/jack-admin kill-server
			'''
		}
		stage('make image') {
			sh '''#!/bin/bash
				 . build/envsetup.sh
				lunch "$RADXA_LUNCH"-userdebug

				# make rk image
				ln -s -f RKTools/linux/Linux_Pack_Firmware/rockdev/ rockdev
				
				rm -rf $RADXA_RELEASE_DIR
				mkdir -p $RADXA_RELEASE_DIR
				
				for hardware in $RADXA_VENDOR;
				do
				    cp -f kernel/$hardware.img kernel/resource.img
				    ./mkimage.sh
				    
				    # make update image
				    cd rockdev
				    ln -s -f Image-$RADXA_LUNCH Image
				    ./mkupdate.sh
				    # make gpt image
				    ./android-gpt.sh
				    cd -
				    
				    # release image
				    commitId=`git -C .repo/manifests rev-parse --short HEAD`
				    #typeset -u hardware_up
				    hardware_up="$hardware"
				    release_name="${hardware_up}-nougat-${RADXA_RELEASE_TIME}_${commitId}"
				    
				    mv rockdev/update.img    rockdev/${release_name}-rkupdate.img
				    mv rockdev/Image/gpt.img rockdev/Image/${release_name}-gpt.img
				    
				    tar czvf $RADXA_RELEASE_DIR/${release_name}-rkupdate.tgz rockdev/${release_name}-rkupdate.img
				    tar czvf $RADXA_RELEASE_DIR/${release_name}-gpt.tgz      rockdev/Image/${release_name}-gpt.img
				    
				    cp -f rockdev/Image/resource.img  $RADXA_RELEASE_DIR/resource_$hardware.img
				    cp -f rockdev/Image/idbloader.img $RADXA_RELEASE_DIR
				    cp -f rockdev/Image/parameter.txt $RADXA_RELEASE_DIR
				    cp -f rockdev/Image/kernel.img    $RADXA_RELEASE_DIR
				    cp -f rockdev/Image/boot.img      $RADXA_RELEASE_DIR
				    cp -f rockdev/Image/uboot.img     $RADXA_RELEASE_DIR
				    cp -f rockdev/Image/trust.img     $RADXA_RELEASE_DIR
				done
			'''
		}*/
		stage('release'){
			String changeNote = ""
			if(currentBuild.changeSets != null){
				for (def logs : currentBuild.changeSets) {
				    def entries = logs.items
				    for (int i = 0; i < entries.length; i++) {
				        def entry = entries[i]
				        changeNote += i + ". " + "${entry.msg}" + "\n"
				    }
				}
			}
			env.RADXA_CHANGE = changeNote
			sh '''#!/bin/bash
				set -xe
                shopt -s nullglob
				commitId=`git -C .repo/manifests rev-parse --short HEAD`
				repo manifest -r -o manifest.xml

				tag=${RADXA_RELEASE_TIME}_${commitId}

                github-release release \
                  --tag "${tag}" \
                  --name "${tag}" \
                  --description "${RADXA_CHANGE}" \
                  --draft
                github-release upload \
                  --tag "${tag}" \
                  --name "manifest" \
                  --file "manifest.xml"

				#for file in $RADXA_RELEASE_DIR/*.tgz; do
	            #    github-release upload \
	            #        --tag "${tag}" \
	            #        --name "$(basename "$file")" \
	            #        --file "$file" &
              	#done
              	#wait
                github-release edit \
                  --tag "${tag}" \
                  --name "${tag}" \
                  --description "${RADXA_CHANGE}"
			'''


			/*script {
		        currentBuild.description = env.RADXA_RELEASE_TIME
	    	}*/
		}
    }
}