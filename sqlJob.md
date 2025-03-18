package com.wellsfargo.ccds.batchfacade.integration.mongo.entities;

import java.time.LocalDateTime;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.core.mapping.Field;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "merchLookup")
public class MerchLookup {
    
    @Id
    private String id;
    
    @Field("batchCd")
    private String batchCd;
    
    @Field("tranCd")
    private String tranCd;
    
    @Field("client")
    private String client;
    
    @Field("merchNo")
    private String merchNo;
    
    @Field("audit")
    private Audit audit;
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class Audit {
        private LocalDateTime createdTs;
        private String createdBy;
    }
}

mongo repo

package com.wellsfargo.ccds.batchfacade.integration.mongo.repositories;

import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookup;

@Repository
public interface MerchLookupRepository extends MongoRepository<MerchLookup, String> {
    // Using built-in methods from MongoRepository
}

MOngo Srvice
package com.wellsfargo.ccds.batchfacade.integration.mongo.service;

import java.util.ArrayList;
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;

import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookup;
import com.wellsfargo.ccds.batchfacade.integration.mongo.repositories.MerchLookupRepository;

import lombok.extern.slf4j.Slf4j;

@Service
@Slf4j
public class MerchLookupMongoService {

    @Autowired
    private MerchLookupRepository merchLookupRepository;
    
    @Value("${batch.merchantnumber.page.size:10}")
    private int pageSize;
    
    /**
     * Returns a list of all pages of merchant lookup data
     * 
     * @return List of pages containing merchant lookup data
     */
    public List<Page<MerchLookup>> getAllPages() {
        log.info("Retrieving all pages of merchant lookup data with page size: {}", pageSize);
        
        List<Page<MerchLookup>> allPages = new ArrayList<>();
        int pageNumber = 0;
        boolean hasMoreData = true;
        
        while (hasMoreData) {
            Pageable pageable = PageRequest.of(pageNumber, pageSize);
            Page<MerchLookup> currentPage = merchLookupRepository.findAll(pageable);
            
            if (currentPage.hasContent()) {
                allPages.add(currentPage);
                pageNumber++;
            } else {
                hasMoreData = false;
            }
        }
        
        log.info("Retrieved {} pages of merchant lookup data", allPages.size());
        return allPages;
    }
    
    /**
     * Gets a specific page of merchant lookup data
     * 
     * @param pageNumber The page number to retrieve (0-based)
     * @return The requested page of merchant lookup data
     */
    public Page<MerchLookup> getPage(int pageNumber) {
        log.info("Retrieving page {} of merchant lookup data", pageNumber);
        Pageable pageable = PageRequest.of(pageNumber, pageSize);
        return merchLookupRepository.findAll(pageable);
    }
    
    /**
     * Calculates the total number of pages based on the total record count and page size
     * 
     * @return Total number of pages
     */
    public int getTotalPages() {
        long totalCount = merchLookupRepository.count();
        int totalPages = (int) Math.ceil((double) totalCount / pageSize);
        
        log.info("Total merchant lookup records: {}, Total pages: {}", totalCount, totalPages);
        return totalPages;
    }
}

Sql Entity
package com.wellsfargo.ccds.batchfacade.integration.sql.entities;

import java.time.LocalDateTime;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Table;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Entity
@Table(name = "MERCHANT_NUMBER_LOOKUP_BATCH")
public class MerchantNumberLookup {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "ID")
    private Long id;
    
    @Column(name = "BATCH_CD")
    private String batchCd;
    
    @Column(name = "TRAN_CD")
    private String tranCd;
    
    @Column(name = "CLIENT")
    private String client;
    
    @Column(name = "MERCH_NO")
    private String merchNo;
    
    @Column(name = "CREATED_TS")
    private LocalDateTime createdTs;
    
    @Column(name = "CREATED_BY")
    private String createdBy;
}

SQL Repo

package com.wellsfargo.ccds.batchfacade.integration.sql.repositories;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import com.wellsfargo.ccds.batchfacade.integration.sql.entities.MerchantNumberLookup;

@Repository
public interface MerchantNumberLookupRepository extends JpaRepository<MerchantNumberLookup, Long> {
    // Using built-in methods from JpaRepository
}

SQL Service

package com.wellsfargo.ccds.batchfacade.integration.sql.service;

import java.util.ArrayList;
import java.util.List;
import java.util.stream.Collectors;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookup;
import com.wellsfargo.ccds.batchfacade.integration.sql.entities.MerchantNumberLookup;
import com.wellsfargo.ccds.batchfacade.integration.sql.repositories.MerchantNumberLookupRepository;

import lombok.extern.slf4j.Slf4j;

@Service
@Slf4j
public class MerchantNumberSqlService {

    @Autowired
    private MerchantNumberLookupRepository merchantNumberLookupRepository;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    /**
     * Saves a batch of merchant lookup records to the SQL database
     * 
     * @param merchLookups List of merchant lookup records from MongoDB
     * @return Number of records saved
     */
    @Transactional
    public int saveBatch(List<MerchLookup> merchLookups) {
        log.info("Saving batch of {} merchant lookup records to SQL database", merchLookups.size());
        
        List<MerchantNumberLookup> sqlEntities = merchLookups.stream()
                .map(this::convertToSqlEntity)
                .collect(Collectors.toList());
        
        List<MerchantNumberLookup> savedEntities = merchantNumberLookupRepository.saveAll(sqlEntities);
        
        log.info("Successfully saved {} merchant lookup records to SQL database", savedEntities.size());
        return savedEntities.size();
    }
    
    /**
     * Executes the stored procedure to refresh the materialized view
     */
    public void refreshMaterializedView() {
        log.info("Executing stored procedure to refresh materialized view");
        jdbcTemplate.execute("CALL REFRESH_MERCHANT_NUMBER_LOOKUP_MV()");
        log.info("Materialized view refresh completed");
    }
    
    /**
     * Converts a MongoDB entity to a SQL entity
     * 
     * @param mongoEntity MongoDB entity
     * @return SQL entity
     */
    private MerchantNumberLookup convertToSqlEntity(MerchLookup mongoEntity) {
        MerchantNumberLookup sqlEntity = new MerchantNumberLookup();
        
        sqlEntity.setBatchCd(mongoEntity.getBatchCd());
        sqlEntity.setTranCd(mongoEntity.getTranCd());
        sqlEntity.setClient(mongoEntity.getClient());
        sqlEntity.setMerchNo(mongoEntity.getMerchNo());
        sqlEntity.setCreatedTs(mongoEntity.getAudit().getCreatedTs());
        sqlEntity.setCreatedBy(mongoEntity.getAudit().getCreatedBy());
        
        return sqlEntity;
    }
}

JOb

package com.wellsfargo.ccds.batchfacade.jobs.impl;

import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.stereotype.Component;

import com.wellsfargo.ccds.batchfacade.integration.message.event.JobEvent;
import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookup;
import com.wellsfargo.ccds.batchfacade.integration.mongo.service.MerchLookupMongoService;
import com.wellsfargo.ccds.batchfacade.integration.sql.service.MerchantNumberSqlService;
import com.wellsfargo.ccds.batchfacade.jobs.JobStrategy;

import lombok.extern.slf4j.Slf4j;

@Component
@Slf4j
public class MerchantNumberJobStrategy implements JobStrategy {

    @Autowired
    private MerchLookupMongoService merchLookupMongoService;
    
    @Autowired
    private MerchantNumberSqlService merchantNumberSqlService;
    
    @Override
    public void execute(JobEvent event) {
        log.info("Starting Merchant Number job with event ID: {}", event.getEventId());
        
        try {
            // Calculate total pages for logging/monitoring
            int totalPages = merchLookupMongoService.getTotalPages();
            log.info("Found {} pages of merchant lookup data to process", totalPages);
            
            if (totalPages == 0) {
                log.info("No merchant lookup data to process. Job completed.");
                return;
            }
            
            int totalProcessed = 0;
            
            // Process each page
            for (int pageNumber = 0; pageNumber < totalPages; pageNumber++) {
                log.info("Processing page {} of {}", pageNumber + 1, totalPages);
                
                // Get data from MongoDB
                Page<MerchLookup> currentPage = merchLookupMongoService.getPage(pageNumber);
                List<MerchLookup> pageData = currentPage.getContent();
                
                if (!pageData.isEmpty()) {
                    // Save data to SQL database
                    int processedCount = merchantNumberSqlService.saveBatch(pageData);
                    totalProcessed += processedCount;
                    
                    log.info("Processed {} records from page {}", processedCount, pageNumber + 1);
                }
            }
            
            // Refresh materialized view if data was processed
            if (totalProcessed > 0) {
                log.info("Refreshing materialized view after processing {} records", totalProcessed);
                merchantNumberSqlService.refreshMaterializedView();
            }
            
            log.info("Merchant Number job completed successfully. Total records processed: {}", totalProcessed);
        } catch (Exception e) {
            log.error("Error executing Merchant Number job: {}", e.getMessage(), e);
            throw new RuntimeException("Failed to execute Merchant Number job", e);
        }
    }
}
