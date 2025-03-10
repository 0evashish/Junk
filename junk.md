package com.wellsfargo.ccds.batchfacade.jobs.impl;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import com.wellsfargo.ccds.batchfacade.integration.message.event.JobEvent;
import com.wellsfargo.ccds.batchfacade.integration.mongo.repositories.MerchLookupRepository;
import com.wellsfargo.ccds.batchfacade.integration.service.MerchantNumberService;
import com.wellsfargo.ccds.batchfacade.jobs.JobStrategy;

import lombok.extern.slf4j.Slf4j;

@Component
@Slf4j
public class MerchantNumberJobStrategy implements JobStrategy {

    @Autowired
    private MerchLookupRepository merchLookupRepository;
    
    @Autowired
    private MerchantNumberService merchantNumberService;
    
    @Value("${batch.merchantnumber.page.size:10}")
    private int pageSize;
    
    @Override
    public void execute(JobEvent event) {
        log.info("Executing Merchant Number job with event: {}", event);
        
        try {
            log.info("Starting to read data from MongoDB in pages of size: {}", pageSize);
            
            // Process data in pages and get total count of processed records
            int totalProcessed = merchantNumberService.processDataInPages(pageSize);
            
            log.info("Successfully processed {} merchant lookup records", totalProcessed);
            
            // After successfully inserting all records, refresh the materialized view
            if (totalProcessed > 0) {
                log.info("Refreshing materialized view after successful data insertion");
                merchantNumberService.refreshMaterializedView();
                log.info("Materialized view refresh completed successfully");
            } else {
                log.info("No records were processed, skipping materialized view refresh");
            }
            
            log.info("Merchant Number job execution completed successfully");
        } catch (Exception e) {
            log.error("Error executing Merchant Number job: {}", e.getMessage(), e);
            throw new RuntimeException("Failed to execute Merchant Number job", e);
        }
    }
}
