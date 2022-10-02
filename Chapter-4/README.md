# AMB Hyperledger Fabric network setup, peer and channel creation

## Cloud9 Env
1. Set up Cloud9 env as in [AMB workshop](https://track-and-trace-blockchain.workshop.aws/)

```
sudo pip install awscli --upgrade
sudo yum install -y jq
aws configure set default.region eu-west-1
```

```
SIZE=${1:-40}
INSTANCEID=$(curl http://169.254.169.254/latest/meta-data//instance-id)
VOLUMEID=$(aws ec2 describe-instances \
  --instance-id $INSTANCEID \
  --query "Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId" \
  --output text)

aws ec2 modify-volume --volume-id $VOLUMEID --size $SIZE
while [ \
  "$(aws ec2 describe-volumes-modifications \
    --volume-id $VOLUMEID \
    --filters Name=modification-state,Values="optimizing","completed" \
    --query "length(VolumesModifications)"\
    --output text)" != "1" ]; do
sleep 1
done

if [ $(readlink -f /dev/xvda) = "/dev/xvda" ]
then
  sudo growpart /dev/xvda 1
  sudo xfs_growfs /dev/xvda1
else
  sudo growpart /dev/nvme0n1 1
  sudo xfs_growfs /dev/nvme0n1p1
fi
```

2. Creat IAM policy _**AmazonManagedBlockchainControlPolicy**_ and role _**ServiceLinkedRoleForAmazonManagedBlockchain**_ - AdministratorAccess to be removed after initial setup
https://catalog.us-east-1.prod.workshops.aws/workshops/ce1e960e-a811-475f-a221-2afcf57e386a/en-US/00-prerequisites/02-iam-configuration

3. Modify Cloud9 IAM role https://catalog.us-east-1.prod.workshops.aws/workshops/ce1e960e-a811-475f-a221-2afcf57e386a/en-US/00-prerequisites/03-attach-machine-role

## Fabric Client setup on Cloud9
https://catalog.us-east-1.prod.workshops.aws/workshops/ce1e960e-a811-475f-a221-2afcf57e386a/en-US/02-set-up-a-fabric-client
1. Create SG _**HLFCliendAndEndpoint**_ with inbound and outbound self-referencing rules
2. Create VPC endpoints to all subnets
3. Add the SG to Cloud9 instance
4. Install packages: from https://docs.aws.amazon.com/managed-blockchain/latest/hyperledger-fabric-dev/get-started-create-client.html
```
sudo yum update -y
sudo yum install jq telnet emacs docker libtool libtool-ltdl-devel git -y
sudo service docker start
sudo usermod -a -G docker ec2-user
```
Restart terminal via ```exit``` to enable usermod, then
```
sudo curl -L \
https://github.com/docker/compose/releases/download/1.20.0/docker-compose-`uname \
-s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod a+x /usr/local/bin/docker-compose
wget https://dl.google.com/go/go1.14.4.linux-amd64.tar.gz
tar -xzf go1.14.4.linux-amd64.tar.gz
sudo mv go /usr/local
```
5. Update ~/.bash_profile with the Orderer endpoints and CA endpoints
6. Proceed with Member name and ID setup, Network ID setup
```
cd
add_line_to_profile_if_not_there() { grep -qxF "$1" .bash_profile || echo "$1" >> .bash_profile; }
export MEMBER_NAME='Alice'
export line="export MEMBER_NAME='$MEMBER_NAME'"
add_line_to_profile_if_not_there "$line"
export line="export NETWORKID=\$(aws managedblockchain list-networks | jq -r '.Networks[] | select(.Name == \"InsurDataNetwork\").Id')"
add_line_to_profile_if_not_there "$line"
export line="export ORDERER=\$(aws managedblockchain get-network --network-id \$NETWORKID | jq -r .Network.FrameworkAttributes.Fabric.OrderingServiceEndpoint)"
add_line_to_profile_if_not_there "$line"
export line="export CASERVICEENDPOINT=\$(aws managedblockchain get-member --network-id \$NETWORKID --member-id \$MEMBERID | jq -r .Member.FrameworkAttributes.Fabric.CaEndpoint)"
add_line_to_profile_if_not_there "$line"
export line="export MEMBERID=\$(aws managedblockchain list-members --network-id \$NETWORKID | jq -r \".Members[] | select(.Name == \\\"\$MEMBER_NAME\\\") | .Id\")"
add_line_to_profile_if_not_there "$line"
export line="export PEERID=\$(aws managedblockchain list-nodes --network-id \$NETWORKID --member-id \$MEMBERID | jq -r \"[.Nodes[] | select(.Status == \\\"AVAILABLE\\\")][0].Id\")"
add_line_to_profile_if_not_there "$line"
export line="export PEERENDPOINT=\$(aws managedblockchain get-node --network-id \$NETWORKID --member-id \$MEMBERID --node-id \$PEERID | jq -r .Node.FrameworkAttributes.Fabric.PeerEndpoint)"
add_line_to_profile_if_not_there "$line"
export line="export CHANNELID='policydata-channel'"
add_line_to_profile_if_not_there "$line"
source ~/.bash_profile
```
7. Set up Fabric CA client
```
mkdir -p /home/ec2-user/go/src/github.com/hyperledger/fabric-ca
cd /home/ec2-user/go/src/github.com/hyperledger/fabric-ca
wget https://github.com/hyperledger/fabric-ca/releases/download/v1.4.7/hyperledger-fabric-ca-linux-amd64-1.4.7.tar.gz
tar -xzf hyperledger-fabric-ca-linux-amd64-1.4.7.tar.gz
```
8. Clone Fabric Samples repo
```
cd /home/ec2-user
git clone --branch v2.2.3 https://github.com/hyperledger/fabric-samples.git
```
9. Create ```docker-compose-cli.yaml``` in the ```/home/ec2-user``` directory:
```
version: '2'
services:
  cli:
    container_name: cli
    image: hyperledger/fabric-tools:2.2
    tty: true
    environment:
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - FABRIC_LOGGING_SPEC=info # Set logging level to debug for more verbose logging
      - CORE_PEER_ID=cli
      - CORE_CHAINCODE_KEEPALIVE=10
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/home/managedblockchain-tls-chain.pem
      - CORE_PEER_LOCALMSPID=$MEMBERID
      - CORE_PEER_MSPCONFIGPATH=/opt/home/admin-msp
      - CORE_PEER_ADDRESS=$PEERENDPOINT
    working_dir: /opt/home
    command: /bin/bash
    volumes:
        - /var/run/:/host/var/run/
        - /home/ec2-user/fabric-samples/chaincode:/opt/gopath/src/github.com/
        - /home/ec2-user:/opt/home
```

and run the HLF CLI container: ```docker-compose -f docker-compose-cli.yaml up -d```

## Enroll the Member Admin
1. Create cert file ```aws s3 cp s3://eu-west-1.managedblockchain/etc/managedblockchain-tls-chain.pem  /home/ec2-user/managedblockchain-tls-chain.pem```
2. Enroll the fabric ca admin via ca client
```
fabric-ca-client enroll \
-u 'https://<admin_username>:<Admin_pwd>@<CA_endpoint>' \
--tls.certfiles /home/ec2-user/managedblockchain-tls-chain.pem -M /home/ec2-user/admin-msp
```
You should see log lines like:
```
2020/05/11 03:29:47 [INFO] Stored client certificate at $HOME/admin-msp/signcerts/cert.pem
2020/05/11 03:29:47 [INFO] Stored root CA certificate at $HOME/admin-msp/cacerts/...
```
Then copy certificates for MSP:
```
cp -r /home/ec2-user/admin-msp/signcerts /home/ec2-user/admin-msp/admincerts
```

## Create Channel
1. Create channel config
```
cat <<EOT > configtx.yaml
################################################################################
#
#   ORGANIZATIONS
#
#   This section defines the organizational identities that can be referenced
#   in the configuration profiles.
#
################################################################################
Organizations:
    # Org1 defines an MSP using the sampleconfig. It should never be used
    # in production but may be used as a template for other definitions.
    - &$MEMBER_NAME
        # Name is the key by which this org will be referenced in channel
        # configuration transactions.
        # Name can include alphanumeric characters as well as dots and dashes.
        Name: $MEMBERID
        # ID is the key by which this org's MSP definition will be referenced.
        # ID can include alphanumeric characters as well as dots and dashes.
        ID: $MEMBERID
        # SkipAsForeign can be set to true for org definitions which are to be
        # inherited from the orderer system channel during channel creation.  This
        # is especially useful when an admin of a single org without access to the
        # MSP directories of the other orgs wishes to create a channel.  Note
        # this property must always be set to false for orgs included in block
        # creation.
        SkipAsForeign: false
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('$MEMBER_NAME.member')"
                # If your MSP is configured with the new NodeOUs, you might
                # want to use a more specific rule like the following:
                # Rule: "OR('$MEMBER_NAME.admin', '$MEMBER_NAME.peer', '$MEMBER_NAME.client')"
            Writers:
                Type: Signature
                Rule: "OR('$MEMBER_NAME.member')"
                # If your MSP is configured with the new NodeOUs, you might
                # want to use a more specific rule like the following:
                # Rule: "OR('$MEMBER_NAME.admin', '$MEMBER_NAME.client')"
            Admins:
                Type: Signature
                Rule: "OR('$MEMBER_NAME.admin')"
        # MSPDir is the filesystem path which contains the MSP configuration.
        MSPDir: /opt/home/admin-msp
        # AnchorPeers defines the location of peers which can be used for
        # cross-org gossip communication. Note, this value is only encoded in
        # the genesis block in the Application section context.
        AnchorPeers:
            - Host: 127.0.0.1
              Port: 7051
################################################################################
#
#   CAPABILITIES
#
#   This section defines the capabilities of fabric network. This is a new
#   concept as of v1.1.0 and should not be utilized in mixed networks with
#   v1.0.x peers and orderers.  Capabilities define features which must be
#   present in a fabric binary for that binary to safely participate in the
#   fabric network.  For instance, if a new MSP type is added, newer binaries
#   might recognize and validate the signatures from this type, while older
#   binaries without this support would be unable to validate those
#   transactions.  This could lead to different versions of the fabric binaries
#   having different world states.  Instead, defining a capability for a channel
#   informs those binaries without this capability that they must cease
#   processing transactions until they have been upgraded.  For v1.0.x if any
#   capabilities are defined (including a map with all capabilities turned off)
#   then the v1.0.x peer will deliberately crash.
#
################################################################################
Capabilities:
    # Channel capabilities apply to both the orderers and the peers and must be
    # supported by both.
    # Set the value of the capability to true to require it.
    # Note that setting a later Channel version capability to true will also
    # implicitly set prior Channel version capabilities to true. There is no need
    # to set each version capability to true (prior version capabilities remain
    # in this sample only to provide the list of valid values).
    Channel: &ChannelCapabilities
        # V2.0 for Channel is a catchall flag for behavior which has been
        # determined to be desired for all orderers and peers running at the v2.0.0
        # level, but which would be incompatible with orderers and peers from
        # prior releases.
        # Prior to enabling V2.0 channel capabilities, ensure that all
        # orderers and peers on a channel are at v2.0.0 or later.
        V2_0: true
    # Orderer capabilities apply only to the orderers, and may be safely
    # used with prior release peers.
    # Set the value of the capability to true to require it.
    Orderer: &OrdererCapabilities
        # V1.1 for Orderer is a catchall flag for behavior which has been
        # determined to be desired for all orderers running at the v1.1.x
        # level, but which would be incompatible with orderers from prior releases.
        # Prior to enabling V2.0 orderer capabilities, ensure that all
        # orderers on a channel are at v2.0.0 or later.
        V2_0: true
    # Application capabilities apply only to the peer network, and may be safely
    # used with prior release orderers.
    # Set the value of the capability to true to require it.
    # Note that setting a later Application version capability to true will also
    # implicitly set prior Application version capabilities to true. There is no need
    # to set each version capability to true (prior version capabilities remain
    # in this sample only to provide the list of valid values).
    Application: &ApplicationCapabilities
        # V2.0 for Application enables the new non-backwards compatible
        # features and fixes of fabric v2.0.
        # Prior to enabling V2.0 orderer capabilities, ensure that all
        # orderers on a channel are at v2.0.0 or later.
        V2_0: true
################################################################################
#
#   CHANNEL
#
#   This section defines the values to encode into a config transaction or
#   genesis block for channel related parameters.
#
################################################################################
Channel: &ChannelDefaults
    # Policies defines the set of policies at this level of the config tree
    # For Channel policies, their canonical path is
    #   /Channel/<PolicyName>
    Policies:
        # Who may invoke the 'Deliver' API
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        # Who may invoke the 'Broadcast' API
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        # By default, who may modify elements at this config level
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
    # Capabilities describes the channel level capabilities, see the
    # dedicated Capabilities section elsewhere in this file for a full
    # description
    Capabilities:
        <<: *ChannelCapabilities
################################################################################
#
#   APPLICATION
#
#   This section defines the values to encode into a config transaction or
#   genesis block for application-related parameters.
#
################################################################################
Application: &ApplicationDefaults
    # Organizations is the list of orgs which are defined as participants on
    # the application side of the network
    Organizations:
    # Policies defines the set of policies at this level of the config tree
    # For Application policies, their canonical path is
    #   /Channel/Application/<PolicyName>
    Policies: &ApplicationDefaultPolicies
        LifecycleEndorsement:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Endorsement:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"

    Capabilities:
        <<: *ApplicationCapabilities
################################################################################
#
#   PROFILES
#
#   Different configuration profiles may be encoded here to be specified as
#   parameters to the configtxgen tool. The profiles which specify consortiums
#   are to be used for generating the orderer genesis block. With the correct
#   consortium members defined in the orderer genesis block, channel creation
#   requests may be generated with only the org member names and a consortium
#   name.
#
################################################################################
Profiles:
    OneOrgChannel:
        <<: *ChannelDefaults
        Consortium: AWSSystemConsortium
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - <<: *$MEMBER_NAME
EOT
```
2. Generate configtx peer block
```
docker exec cli configtxgen \
-outputCreateChannelTx /opt/home/$CHANNELID.pb \
-profile OneOrgChannel -channelID $CHANNELID \
--configPath /opt/home/
```
3. Create the channel
```
docker exec cli peer channel create -c $CHANNELID \
-f /opt/home/$CHANNELID.pb -o $ORDERER \
--cafile /opt/home/managedblockchain-tls-chain.pem --tls
```
4. Join your peer node to the channel
```
docker exec cli peer channel join -b $CHANNELID.block \
-o $ORDERER --cafile /opt/home/managedblockchain-tls-chain.pem --tls
```