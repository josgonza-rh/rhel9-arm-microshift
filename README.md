# How to create your Red Hat Enterprise Linux 9 for ARM 64 image with MicroShift using Image Builder

Being involved in a demo of `Red Hat Enterprise Linux for Edge`, `MicroShift` and `ImageBuilder` and due to problems with the network driver in `RHEL 8.x` for `Raspberry PI 4` the need arose to have a custom image in version 9 (where the Ethernet network driver does work, but not the wireless). Since there is no RHEL 9 AMI in the AWS marketplace (at the time of writing is a Beta version), I have documented the steps to be performed in my git repo: [rhel9-arm-ami](https://github.com/josgonza-rh/rhel9-arm-ami).

Assuming that all the work in the [rhel9-arm-ami](https://github.com/josgonza-rh/rhel9-arm-ami) repo has been done (creation of the AMI), I have documented the steps to follow in order to build your RHEL 9 for Edge ARM 64 custom image (with `MicroShift`).

## Building RHEL for Edge with Image Builder

Let's look at using [Image Builder](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9-beta/html/composing_a_customized_rhel_system_image/index) to create a RHEL 9 for Edge with MicroShift `raw` image.

Image Builder can create installer media that embeds the OSTree commit. This is perfect for disconnected environments that require an `ISO`, thumb drive, or some form of media that doesn't depend on network access. In our case, we are going to use the `raw` format.

### Prerequisites

In order to perform all the steps as documented, we will need an AWS account and access to the EC2 [Graviton](https://aws.amazon.com/ec2/graviton/) service (ARM compute).

Of course, although the purpose is to show the whole process based on `ARM` architecture, depending on the computing capacity you have on your laptop or server, you can make use of `KVM` and virtualization and avoid all the Amazon stuff.

> ![WARNING](images/warning-icon.png) **WARNING**: As mentioned above, it is important to have all the work discussed in the [rhel9-arm-ami](https://github.com/josgonza-rh/rhel9-arm-ami) repo. If not, the following steps are likely to vary and may not work correctly.

### Launch the EC2 Graviton instance

We are going to launch a `Graviton` instance, which is what AWS calls EC2 instances with `ARM` architecture.

1. Create ssh key-pair (optional):

    ```bash
    ssh-keygen -b 4096
    ```

    > ![NOTE](images/note-icon.png) **NOTE**: Optional. You can create new ones or use existing ones, necessary to be able to connect to the EC2 instance.

2. Import key:

    ```bash
    aws ec2 import-key-pair --key-name "image_builder" --public-key-material fileb://~/.ssh/image_builder.pub
    ```

3. Start the instance:

    ```bash
    aws ec2 run-instances \
        --image-id ami-0f890ee1070bb67cf \
        --instance-type t4g.medium \
        --subnet-id subnet-0f0b77ba349b21e50 \
        --security-group-ids sg-047e6a8ef2b4edb23 \
        --associate-public-ip-address \
        --key-name image_builder \
        --block-device-mappings "[{\"DeviceName\":\"/dev/xvda\",\"Ebs\":{\"VolumeSize\":60,\"DeleteOnTermination\":true}}]" \
        --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=image-builder}]'
    ```

    > ![WARNING](images/warning-icon.png) **WARNING**: `image-id` is the ID of the custom AMI I created (is not an existing or valid ID within the AWS public marketplace)

### Install Image Builder

Step by step:

1. Register the system

    ```bash
    subscription-manager register --username your@user.com --password your_password
    subscription-manager role --set="Red Hat Enterprise Linux Server"
    subscription-manager service-level --set="Self-Support"
    subscription-manager usage --set="Development/Test"
    ```

2. Install Image Builder

    ```bash
    yum install -y osbuild-composer
    systemctl enable --now osbuild-composer.socket
    ```

3. Install composer

    ```bash
    yum install -y composer-cli
    ```

4. Enable web console

    ```bash
    yum install -y cockpit-composer
    systemctl enable --now cockpit.socket
    ```

5. Install conteainer tools

    ```bash
    yum module install -y container-tools
    ```

> ![TIP](images/tip-icon.png) **TIP**: Step 4 and 5 are optional, but help to have a complete environment ready for any kind of scenario. The web console makes it easy to generate/download the image and the container tools make it possible to control the `OSTree Commit` container image via `podman` / `skopeo`.

### MicroShift & Image Builder

We need to configure a [repository override](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9-beta/html-single/composing_a_customized_rhel_system_image/index#overriding-a-system-repository_managing-repositories) to be able to add custom repositories (or repositories that are different from the default repositories):

1. Create the directory

    ```bash
    mkdir -p /etc/osbuild-composer/repositories
    ```

2. Copy the file [rhel-90.json](utils/rhel-90.json) to the directory

    ```bash
    cp utils/rhel-90.json /etc/osbuild-composer/repositories/
    ```

3. Add new repositories to the server

    ```bash
    composer-cli sources add utils/transmission.toml
    composer-cli sources add utils/microshift.toml
    ```

    > ![INFO](images/info-icon.png) **INFO**: A new source repository can be added by creating a TOML file with the details of the repository and add it to the server. We need to add the following repos to access the `MicroShift` packages:
    - [transmission.toml](utils/transmission.toml)
    - [microshift.toml](utils/microshift.toml)

4. Reboot `Image Builder` to detect the changes

    ```bash
    systemctl restart osbuild-composer.service
    ```

5. Build the image:

    ```bash
    composer-cli blueprints push utils/microshift-blueprint-v0.0.1.toml
    ```

    > ![TIP](images/tip-icon.png) **TIP**: Within the supported customizations within the blueprint I would recommend adding at least the public ssh key. For more information see the documentation: [Supported Image Customizations](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9-beta/html-single/composing_a_customized_rhel_system_image/index#image-customizations_creating-system-images-with-composer-command-line-interface)

### Installing Images on Raspberry Pi

Once we have our image created, we can download it by using `composer-cli` or the `cockpit` web console and burn it into our `Raspberry`.

Here you have multiple choices:
- [etcher](https://www.balena.io/etcher/)
- [Raspberry Pi Imager](https://www.raspberrypi.com/documentation/computers/getting-started.html#using-raspberry-pi-imager)
- [On Linux](https://www.raspberrypi.com/documentation/computers/getting-started.html#installing-images-on-linux)

## Recommended links

As you have seen, this is a very specific scenario / use case, but if you want to go deeper into topics such as `Edge`, `Image Builder` and/or `MicroShift`, I recommend the following links:

- [Building RHEL for Edge with Image Builder](https://github.com/osbuild/rhel-for-edge-demo)
- [RHEL for Edge deployment types](https://github.com/luisarizmendi/rhel-edge-quickstart): from my buddy @luisarizmendi with very good scripts
- [Introducing MicroShift](https://next.redhat.com/2022/01/19/introducing-microshift/): amazing post regarding MicroShift and its role in the Edge from my pal @oglok
