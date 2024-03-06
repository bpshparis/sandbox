https://www.vstellar.com/2017/07/25/automating-esxi-deployment-using-pxe-boot-and-kickstart/



https://www.unixmen.com/install-pxe-server-centos-7/







WEB_SERVER_ISO_URL="http://192.168.1.33:81"
RHCOS_ISO_FILE="rhcos-installer.x86_64.iso"
ISO_PATH="/media/iso"
RW_ISO_PATH="/media/isorw"
TEST_ISO_PATH="/media/test"



[ ! -d $ISO_PATH ] && sudo mkdir $ISO_PATH 

while [ ! -z "$(ls -A $ISO_PATH)" ]; do sudo umount $ISO_PATH; sleep 2; done

sudo mount -o loop $RHCOS_ISO_FILE $ISO_PATH

[ ! -d $RW_ISO_PATH ] && sudo mkdir $RW_ISO_PATH || sudo rm -rf $RW_ISO_PATH/*



while [ ! -z "$(ls -A $ISO_PATH)" ]; do sudo umount $ISO_PATH; sleep 2; done
sudo rmdir $ISO_PATH
sudo rm -rf $RW_ISO_PATH



[ ! -d $TEST_ISO_PATH ] && sudo mkdir $TEST_ISO_PATH

for iso in $(ls bs-ocp.iso); do
    echo $iso
    sudo mount -o loop $iso $TEST_ISO_PATH
   cat $TEST_ISO_PATH/isolinux/isolinux.cfg
    sleep 2
    sudo umount $TEST_ISO_PATH
done

while [ ! -z "$(ls -A $TEST_ISO_PATH)" ]; do sudo umount $TEST_ISO_PATH; sleep 2; done

sudo rmdir $TEST_ISO_PATH

