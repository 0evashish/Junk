// 1. Common Audit Class
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

// 2. MerchLookupRap Entity Class
package com.wellsfargo.ccds.batchfacade.integration.mongo.entities;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Getter;
import lombok.Setter;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection = "merchLookupRap")
@Getter
@Setter
public class MerchLookupRap {
    
    @Id
    @JsonProperty("_id")
    private String id;
    
    @JsonProperty("acctPrefix")
    private String acctPrefix;
    
    @JsonProperty("orgAppId")
    private String orgAppId;
    
    @JsonProperty("merchNbr")
    private String merchNbr;
    
    @JsonProperty("authTrans")
    private String authTrans;
    
    @JsonProperty("audit")
    private Audit audit;
}

// 3. SuperCheckLookup Entity Class
package com.wellsfargo.ccds.batchfacade.integration.mongo.entities;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Getter;
import lombok.Setter;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection = "superCheckLookup")
@Getter
@Setter
public class SuperCheckLookup {
    
    @Id
    @JsonProperty("_id")
    private String id;
    
    @JsonProperty("shareDraftPrefix")
    private String shareDraftPrefix;
    
    @JsonProperty("acctPrefix")
    private String acctPrefix;
    
    @JsonProperty("merchNbr")
    private String merchNbr;
    
    @JsonProperty("typeCd")
    private String typeCd;
    
    @JsonProperty("audit")
    private Audit audit;
}

// 4. MerchLookupRap Repository Interface
package com.wellsfargo.ccds.batchfacade.integration.mongo.repositories;

import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookupRap;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface MerchLookupRapRepository extends MongoRepository<MerchLookupRap, String> {
    // MongoRepository provides built-in methods for CRUD operations
}

// 5. SuperCheckLookup Repository Interface
package com.wellsfargo.ccds.batchfacade.integration.mongo.repositories;

import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.SuperCheckLookup;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface SuperCheckLookupRepository extends MongoRepository<SuperCheckLookup, String> {
    // MongoRepository provides built-in methods for CRUD operations
}

// 6. RapSyncService Class
package com.wellsfargo.ccds.batchfacade.services;

import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookupRap;
import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.SuperCheckLookup;
import com.wellsfargo.ccds.batchfacade.integration.mongo.repositories.MerchLookupRapRepository;
import com.wellsfargo.ccds.batchfacade.integration.mongo.repositories.SuperCheckLookupRepository;
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
public class RapSyncService {
    
    private static final Logger log = LoggerFactory.getLogger(RapSyncService.class);
    
    @Autowired
    private MerchLookupRapRepository merchLookupRapRepository;
    
    @Autowired
    private SuperCheckLookupRepository superCheckLookupRepository;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Value("${batch.page.size}")
    private int pageSize;
    
    /**
     * Retrieves a specific page of MerchLookupRap data from MongoDB
     * 
     * @param pageNumber The page number to retrieve (0-based)
     * @return Page of MerchLookupRap data
     */
    public Page<MerchLookupRap> getMerchLookupRapPage(int pageNumber) {
        log.info("Fetching MerchLookupRap page {} with size {}", pageNumber, pageSize);
        Pageable pageable = PageRequest.of(pageNumber, pageSize);
        return merchLookupRapRepository.findAll(pageable);
    }
    
    /**
     * Calculates the total number of pages for MerchLookupRap based on record count and page size
     * 
     * @return Total number of pages
     */
    public int getTotalMerchLookupRapPages() {
        long totalRecords = merchLookupRapRepository.count();
        log.info("Total MerchLookupRap records in MongoDB: {}", totalRecords);
        return (int) Math.ceil((double) totalRecords / pageSize);
    }
    
    /**
     * Retrieves a specific page of SuperCheckLookup data from MongoDB
     * 
     * @param pageNumber The page number to retrieve (0-based)
     * @return Page of SuperCheckLookup data
     */
    public Page<SuperCheckLookup> getSuperCheckLookupPage(int pageNumber) {
        log.info("Fetching SuperCheckLookup page {} with size {}", pageNumber, pageSize);
        Pageable pageable = PageRequest.of(pageNumber, pageSize);
        return superCheckLookupRepository.findAll(pageable);
    }
    
    /**
     * Calculates the total number of pages for SuperCheckLookup based on record count and page size
     * 
     * @return Total number of pages
     */
    public int getTotalSuperCheckLookupPages() {
        long totalRecords = superCheckLookupRepository.count();
        log.info("Total SuperCheckLookup records in MongoDB: {}", totalRecords);
        return (int) Math.ceil((double) totalRecords / pageSize);
    }
    
    /**
     * Inserts MerchLookupRap records into SQL database
     * 
     * @param merchLookupRaps List of MerchLookupRap records to insert
     */
    public void insertMerchLookupRapIntoSql(List<MerchLookupRap> merchLookupRaps) {
        log.info("Inserting {} MerchLookupRap records into SQL database", merchLookupRaps.size());
        
        for (MerchLookupRap merchLookupRap : merchLookupRaps) {
            jdbcTemplate.update(
                "INSERT INTO MERCH_LOOKUP_RAP (ACCT_PREFIX, ORG_APP_ID, MERCH_NBR, AUTH_TRANS, CREATED_TS, CREATED_BY) " +
                "VALUES (?, ?, ?, ?, ?, ?)",
                merchLookupRap.getAcctPrefix(),
                merchLookupRap.getOrgAppId(),
                merchLookupRap.getMerchNbr(),
                merchLookupRap.getAuthTrans(),
                merchLookupRap.getAudit().getCreatedTs(),
                merchLookupRap.getAudit().getCreatedBy()
            );
        }
        
        log.info("MerchLookupRap SQL insertion completed successfully");
    }
    
    /**
     * Inserts SuperCheckLookup records into SQL database
     * 
     * @param superCheckLookups List of SuperCheckLookup records to insert
     */
    public void insertSuperCheckLookupIntoSql(List<SuperCheckLookup> superCheckLookups) {
        log.info("Inserting {} SuperCheckLookup records into SQL database", superCheckLookups.size());
        
        for (SuperCheckLookup superCheckLookup : superCheckLookups) {
            jdbcTemplate.update(
                "INSERT INTO SUPER_CHECK_LOOKUP (SHARE_DRAFT_PREFIX, ACCT_PREFIX, MERCH_NBR, TYPE_CD, CREATED_TS, CREATED_BY) " +
                "VALUES (?, ?, ?, ?, ?, ?)",
                superCheckLookup.getShareDraftPrefix(),
                superCheckLookup.getAcctPrefix(),
                superCheckLookup.getMerchNbr(),
                superCheckLookup.getTypeCd(),
                superCheckLookup.getAudit().getCreatedTs(),
                superCheckLookup.getAudit().getCreatedBy()
            );
        }
        
        log.info("SuperCheckLookup SQL insertion completed successfully");
    }
    
    /**
     * Executes stored procedure to update materialized views
     */
    public void executeStoredProcedure() {
        log.info("Executing stored procedure to update RAP sync materialized views");
        jdbcTemplate.execute("CALL UPDATE_RAP_SYNC_MATERIALIZED_VIEWS()");
        log.info("Stored procedure execution completed");
    }
}

// 7. RapSyncJobStrategy Implementation
package com.wellsfargo.ccds.batchfacade.jobs.impl;

import com.wellsfargo.ccds.batchfacade.integration.message.event.JobEvent;
import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookupRap;
import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.SuperCheckLookup;
import com.wellsfargo.ccds.batchfacade.jobs.JobStrategy;
import com.wellsfargo.ccds.batchfacade.services.RapSyncService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.stereotype.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Component
public class RapSyncJobStrategy implements JobStrategy {

    private static final Logger log = LoggerFactory.getLogger(RapSyncJobStrategy.class);
    
    @Autowired
    private RapSyncService rapSyncService;
    
    @Override
    public void execute(JobEvent event) {
        log.info("Executing RAP Sync job...");
        
        try {
            // Process MerchLookupRap collection
            processMerchLookupRap();
            
            // Process SuperCheckLookup collection
            processSuperCheckLookup();
            
            // Execute stored procedure to update materialized views
            log.info("Executing stored procedure to update materialized views");
            rapSyncService.executeStoredProcedure();
            
            log.info("RAP Sync job completed successfully");
        } catch (Exception e) {
            log.error("Error executing RAP Sync job", e);
            throw new RuntimeException("Failed to execute RAP Sync job", e);
        }
    }
    
    /**
     * Process MerchLookupRap collection data
     */
    private void processMerchLookupRap() {
        // Get total number of pages
        int totalPages = rapSyncService.getTotalMerchLookupRapPages();
        log.info("Total MerchLookupRap pages to process: {}", totalPages);
        
        // Process each page
        for (int pageNum = 0; pageNum < totalPages; pageNum++) {
            log.info("Processing MerchLookupRap page {} of {}", pageNum + 1, totalPages);
            
            // Get current page of data
            Page<MerchLookupRap> page = rapSyncService.getMerchLookupRapPage(pageNum);
            
            // Insert data into SQL DB
            rapSyncService.insertMerchLookupRapIntoSql(page.getContent());
            
            log.info("Successfully processed MerchLookupRap page {} with {} records", 
                pageNum + 1, page.getNumberOfElements());
        }
    }
    
    /**
     * Process SuperCheckLookup collection data
     */
    private void processSuperCheckLookup() {
        // Get total number of pages
        int totalPages = rapSyncService.getTotalSuperCheckLookupPages();
        log.info("Total SuperCheckLookup pages to process: {}", totalPages);
        
        // Process each page
        for (int pageNum = 0; pageNum < totalPages; pageNum++) {
            log.info("Processing SuperCheckLookup page {} of {}", pageNum + 1, totalPages);
            
            // Get current page of data
            Page<SuperCheckLookup> page = rapSyncService.getSuperCheckLookupPage(pageNum);
            
            // Insert data into SQL DB
            rapSyncService.insertSuperCheckLookupIntoSql(page.getContent());
            
            log.info("Successfully processed SuperCheckLookup page {} with {} records", 
                pageNum + 1, page.getNumberOfElements());
        }
    }
}
