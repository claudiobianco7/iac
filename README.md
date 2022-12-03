        stage('Create ChangeSet'){
            steps{
                echo "################################"
                echo "####### Create ChangeSet #######"
                echo "################################"

                echo "Updating parameters file..."
                sh (""" sed -i 's/index.zip/index-${BUILD_ID}.zip/' ${PARAMETERS_DEV_FILE} """)
                sh (""" cat ${PARAMETERS_DEV_FILE} """)

                sh (""" cat ${TEMPLATE_FILE} """)

//                sh("""aws cloudformation create-change-set \
//                    --stack-name ${STACK_NAME_DEV} \
//                    --change-set-name ${STACK_NAME_DEV}-${BUILD_ID} \
//                    --template-body file://${TEMPLATE_FILE} \
//                    --parameters file://${PARAMETERS_DEV_FILE} \
//                    --description ${AMI_ID} \
//                    --capabilities CAPABILITY_NAMED_IAM """)
            }
        }
