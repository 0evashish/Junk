// 1. Audit Class (if not already defined elsewhere)
package com.wellsfargo.ccds.batchfacade.integration.mongo.entities;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Getter;
import lombok.Setter;
import java.util.Date;

@Getter
@Setter
public class Audit {
    @JsonProperty("createdTs")
    private Date createdTs;
    
    @JsonProperty("createdBy")
    private String createdBy;
}

// 2. MerchLookup Entity Class
package com.wellsfargo.ccds.batchfacade.integration.mongo.entities;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Getter;
import lombok.Setter;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection = "merchLookup")
@Getter
@Setter
public class MerchLookup {
    
    @Id
    @JsonProperty("_id")
    private String id;
    
    @JsonProperty("batchCd")
    private String batchCd;
    
    @JsonProperty("tranCd")
    private String tranCd;
    
    @JsonProperty("client")
    private String client;
    
    @JsonProperty("system")
    private String system;
    
    @JsonProperty("merchNo")
    private String merchNo;
    
    @JsonProperty("audit")
    private Audit audit;
}

// 3. MerchLookup Repository Interface
package com.wellsfargo.ccds.batchfacade.integration.mongo.repositories;

import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookup;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface MerchLookupRepository extends MongoRepository<MerchLookup, String> {
    // MongoRepository provides built-in methods for CRUD operations
}

// 4. MerchantNumberService Class
package com.wellsfargo.ccds.batchfacade.services;

import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookup;
import com.wellsfargo.ccds.batchfacade.integration.mongo.repositories.MerchLookupRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.List;

@Service
public class MerchantNumberService {
    
    private static final Logger log = LoggerFactory.getLogger(MerchantNumberService.class);
    
    @Autowired
    private MerchLookupRepository merchLookupRepository;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Value("${batch.page.size}")
    private int pageSize;
    
    /**
     * Retrieves a specific page of merchant lookup data from MongoDB
     * 
     * @param pageNumber The page number to retrieve (0-based)
     * @return Page of merchant lookup data
     */
    public Page<MerchLookup> getMerchLookupPage(int pageNumber) {
        log.info("Fetching page {} with size {}", pageNumber, pageSize);
        Pageable pageable = PageRequest.of(pageNumber, pageSize);
        return merchLookupRepository.findAll(pageable);
    }
    
    /**
     * Calculates the total number of pages based on record count and page size
     * 
     * @return Total number of pages
     */
    public int getTotalPages() {
        long totalRecords = merchLookupRepository.count();
        log.info("Total records in MongoDB: {}", totalRecords);
        return (int) Math.ceil((double) totalRecords / pageSize);
    }
    
    /**
     * Inserts merchant lookup records into SQL database
     * 
     * @param merchLookups List of merchant lookup records to insert
     */
    public void insertIntoSql(List<MerchLookup> merchLookups) {
        log.info("Inserting {} records into SQL database", merchLookups.size());
        
        for (MerchLookup merchLookup : merchLookups) {
            jdbcTemplate.update(
                "INSERT INTO MERCHANT_NUMBER_LOOKUP_BATCH (BATCH_CD, TRAN_CD, CLIENT, SYSTEM, MERCH_NO, CREATED_TS, CREATED_BY) " +
                "VALUES (?, ?, ?, ?, ?, ?, ?)",
                merchLookup.getBatchCd(),
                merchLookup.getTranCd(),
                merchLookup.getClient(),
                merchLookup.getSystem(),
                merchLookup.getMerchNo(),
                merchLookup.getAudit().getCreatedTs(),
                merchLookup.getAudit().getCreatedBy()
            );
        }
        
        log.info("SQL insertion completed successfully");
    }
    
    /**
     * Executes stored procedure to update materialized view
     */
    public void executeStoredProcedure() {
        log.info("Executing stored procedure to update materialized view");
        jdbcTemplate.execute("CALL UPDATE_MERCHANT_NUMBER_MATERIALIZED_VIEW()");
        log.info("Stored procedure execution completed");
    }
}

// 5. MerchantNumberJobStrategy Implementation
package com.wellsfargo.ccds.batchfacade.jobs.impl;

import com.wellsfargo.ccds.batchfacade.integration.message.event.JobEvent;
import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookup;
import com.wellsfargo.ccds.batchfacade.jobs.JobStrategy;
import com.wellsfargo.ccds.batchfacade.services.MerchantNumberService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.stereotype.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Component
public class MerchantNumberJobStrategy implements JobStrategy {

    private static final Logger log = LoggerFactory.getLogger(MerchantNumberJobStrategy.class);
    
    @Autowired
    private MerchantNumberService merchantNumberService;
    
    @Override
    public void execute(JobEvent event) {
        log.info("Executing Merchant Number job...");
        
        try {
            // Get total number of pages
            int totalPages = merchantNumberService.getTotalPages();
            log.info("Total pages to process: {}", totalPages);
            
            // Process each page
            for (int pageNum = 0; pageNum < totalPages; pageNum++) {
                log.info("Processing page {} of {}", pageNum + 1, totalPages);
                
                // Get current page of data
                Page<MerchLookup> page = merchantNumberService.getMerchLookupPage(pageNum);
                
                // Insert data into SQL DB
                merchantNumberService.insertIntoSql(page.getContent());
                
                log.info("Successfully processed page {} with {} records", 
                    pageNum + 1, page.getNumberOfElements());
            }
            
            // Execute stored procedure to update materialized view
            log.info("Executing stored procedure to update materialized view");
            merchantNumberService.executeStoredProcedure();
            
            log.info("Merchant Number job completed successfully");
        } catch (Exception e) {
            log.error("Error executing Merchant Number job", e);
            throw new RuntimeException("Failed to execute Merchant Number job", e);
        }
    }
}
