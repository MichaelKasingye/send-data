```javaScript
  public async getAllFiles(
        lawFirmId: any,
        page?: any,
        pageSize?: any,
        searchTerms?: any,
        otherQueryParams?: Record<string, any>
    ) {
 

        try {
            const query = this.filesRepository.createQueryBuilder('files');
    
            query.where('files.lawFirmId = :lawFirmId', { lawFirmId })
                .leftJoinAndSelect('files.fileStage', 'fileStage')
                .leftJoinAndSelect('fileStage.substages', 'fileSubStage')
                .leftJoinAndSelect('files.client', 'client')
                .leftJoinAndSelect('files.lawFirm', 'lawFirm')
                .leftJoinAndSelect('files.disbursement', 'disbursement')
                // .leftJoinAndSelect('files.assignees', 'user')
                .orderBy('files.dateOpened', 'DESC')
                .select([
                    'files.id',
                    'files.customFileId',
                    'files.fileNumber',
                    'files.fileName',
                    'files.description',
                    'files.fileType',
                    'files.clientCategory',
                    'files.fileCategory',
                    'files.conflictCheck',
                    'files.client',
                    'files.dateOpened',
                    'files.dateClosed',
                    'files.filesStatus',
                    'files.practiceArea',
                    'files.leadPartners',
                    'files.partners',
                    'files.assignees',
                    'files.court',
                    'files.judge',
                    'client.id',
                    'client.firstName',
                    'client.lastName',
                    'client.name',
                    'client.businessName',
                    'lawFirm.id',
                    'fileStage.id',
                    'fileStage.stage',
                    'fileSubStage.id',
                    'fileSubStage.stageTitle',
                    'fileSubStage.description',
                    'fileSubStage.date',
                    'fileSubStage.createdBy',
                    'fileSubStage.updatedBy',
                    'disbursement.disbursementFees',
                    'disbursement.disbursementAmount',
                    'disbursement.amountUsed',
                    'disbursement.amountRemaining',
                ]);
    
            // Handle search term
            if (searchTerms && searchTerms.trim().length > 0) {
                query.andWhere('(files.fileName ILIKE :search)', {
                    search: `%${searchTerms}%`
                });
            }
let filteredFiles = [];
    
            // Handle additional filters
            if (otherQueryParams) {
                let queriedFilteredFiles = [];
                // Object.entries(otherQueryParams).forEach(async ([key, value]) => {
                    for (const [key, value] of Object.entries(otherQueryParams)) {
                        let filteredFilesAssignees = [];
                    if (value && key !== 'page' && key !== 'pageSize' && key !== 'partners' && key !== 'practiceArea' && key !== 'dateClosed' && key !== 'dateOpened') {
                        console.log('key', key);
                        query.andWhere(`files.${key} ILIKE :${key}`, {
                            [key]: `%${value}%`
                        });
                    }

                    if (key === 'partners' && value !== undefined) {
                   
                        query.andWhere(`
                            EXISTS (
                                SELECT 1 
                                FROM "user" u
                                WHERE 
                                    CONCAT(u."firstName", ' ', u."lastName") ILIKE :partnerName
                                    AND POSITION(u.id::text IN files.partners) > 0
                            )
                        `, {
                            partnerName: `%${value}%`,
                        });
                    }


                    if (key === 'assignees' && value !== undefined) {
                        const trimmedValue = value.trim();
                        
                        // Step 1: Find the user by name
                        const user = await this.userRepository
                            .createQueryBuilder('user')
                            .where(
                                "(user.firstName ILIKE :searchTerm OR user.lastName ILIKE :searchTerm OR CONCAT(user.firstName, ' ', user.lastName) ILIKE :fullSearchTerm)",
                                {
                                    searchTerm: `%${trimmedValue}%`,
                                    fullSearchTerm: `%${trimmedValue}%`
                                }
                            )
                            .getOne();
                        
                        if (!user?.id) {
                            throw new Error('User not found');
                        }
                        
                        const userId = user.id.toString();
                        console.log('Found user with ID:', userId);
                        
                        // Step 2: Get all files
                        // const allFiles = await this.filesRepository.find();
                        const query = this.filesRepository.createQueryBuilder('files');
    
                        query.where('files.lawFirmId = :lawFirmId', { lawFirmId })
                        .leftJoinAndSelect('files.fileStage', 'fileStage')
                        .leftJoinAndSelect('fileStage.substages', 'fileSubStage')
                        .leftJoinAndSelect('files.client', 'client')
                        .leftJoinAndSelect('files.lawFirm', 'lawFirm')
                        .leftJoinAndSelect('files.disbursement', 'disbursement')
                        // .leftJoinAndSelect('files.assignees', 'user')
                        .orderBy('files.dateOpened', 'DESC')
                        .select([
                            'files.id',
                            'files.customFileId',
                            'files.fileNumber',
                            'files.fileName',
                            'files.description',
                            'files.fileType',
                            'files.clientCategory',
                            'files.fileCategory',
                            'files.conflictCheck',
                            'files.client',
                            'files.dateOpened',
                            'files.dateClosed',
                            'files.filesStatus',
                            'files.practiceArea',
                            'files.leadPartners',
                            'files.partners',
                            'files.assignees',
                            'files.court',
                            'files.judge',
                            'client.id',
                            'client.firstName',
                            'client.lastName',
                            'client.name',
                            'client.businessName',
                            'lawFirm.id',
                            'fileStage.id',
                            'fileStage.stage',
                            'fileSubStage.id',
                            'fileSubStage.stageTitle',
                            'fileSubStage.description',
                            'fileSubStage.date',
                            'fileSubStage.createdBy',
                            'fileSubStage.updatedBy',
                            'disbursement.disbursementFees',
                            'disbursement.disbursementAmount',
                            'disbursement.amountUsed',
                            'disbursement.amountRemaining',
                        ]);


                            const allFiles = await query.getMany();
                                                    console.log(`Total files before filtering: ${allFiles.length}`);
                        
                        // Step 3: Filter files manually
                        const filteredFileIds = allFiles
                            .filter((file:any) => {
                                // Check if assignees exists and is not empty
                                if (!file.assignees) return false;
                                
                                // Parse the assignees string into an array of IDs
                                const assigneeIds = Array.isArray(file.assignees) 
                                    ? file.assignees 
                                    : file.assignees.split(',');
                                
                                // Check if our user ID is in the assignees array
                                return assigneeIds.includes(userId);
                            })
                            .map(file => file);
                        
                        console.log(`Files with user ${userId} as assignee: ${filteredFileIds.length}`);
                        // console.log('Filtered file IDs:', filteredFileIds);
                        
                        
                        // Step 4: Add condition to query
                        if (filteredFileIds.length > 0) {
                            query.andWhere('files.id IN (:...fileIds)', { fileIds: filteredFileIds });
                        } else {
                            // No files found with this user, so return an empty result
                            query.andWhere('1 = 0'); // Always false condition
                        }

                        const valueFiles = await query.getMany();
                        // console.log('Filtered file IDs:', filteredFileIds);

                        console.log('Filtered file IDs valueFiles:', valueFiles);
                        filteredFiles.push(...filteredFileIds);
                        // return filteredFileIds
                    }
                
                    if (key === 'practiceArea' && value !== undefined) {
                        query.andWhere(`
                            EXISTS (
                                SELECT 1 
                                FROM "practice_areas" p
                                WHERE 
                                    p."practiceAreaName" ILIKE :namePracticeArea
                                    AND (
                                        files."practiceArea" = p.id::text OR
                                        files."practiceArea" LIKE p.id::text || ',%' OR
                                        files."practiceArea" LIKE '%,' || p.id::text || ',%' OR
                                        files."practiceArea" LIKE '%,' || p.id::text
                                    )
                            )
                        `, {
                            namePracticeArea: `%${value}%`,
                        });
                    }

                    if (key === 'dateOpened' && value !== undefined) {
                        console.log('dateOpened-value', value);

                        query.andWhere(`DATE(files.dateOpened) = :openDate`, {
                            openDate: value,
                        });
                    }

                    if (key === 'dateClosed' && value !== undefined) {
                        console.log('dateClosed-value', value);

                        query.andWhere(`DATE(files.dateClosed) = :closeDate`, {
                            closeDate: value,
                        });
                    }
                    // console.log("filteredFiles---->*",filteredFiles);
                    // queriedFilteredFiles.push(...filteredFilesAssignees);
                }
            // );
            // filteredFiles.push(...queriedFilteredFiles);
                // console.log('queriedFilteredFiles',queriedFilteredFiles);
                
            }
    console.log('filteredFiles',filteredFiles);
    
            // Handle pagination
            const limit = pageSize ? Number(pageSize) : 10;
            const currentPage = page ? Number(page) : 1;
    
            query.take(limit).skip((currentPage - 1) * limit);
            // console.log('filteredFiles....',filteredFiles);
            
            const [files, total] = await query.getManyAndCount();

            if(filteredFiles.length >0){
                return { files:filteredFiles, total:filteredFiles.length };

            }

            return { files, total };
        } catch (error) {
            Logger.error('Error occurred while fetching files:', error);
            throw error;
        }
    }
```
