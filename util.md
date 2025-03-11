// MongoConnectionTest.java
package com.wellsfargo.ccds.batchfacade.util;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.stereotype.Component;

import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookup;
import com.wellsfargo.ccds.batchfacade.integration.mongo.repositories.MerchLookupRepository;

import java.util.List;

/**
 * Utility class to quickly test MongoDB connection and data retrieval
 */
@Component
public class MongoConnectionTest {
    
    private static final Logger log = LoggerFactory.getLogger(MongoConnectionTest.class);
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    @Autowired
    private MerchLookupRepository merchLookupRepository;
    
    /**
     * Simple method to test MongoDB connection and data retrieval
     * @return true if connection and data retrieval are successful
     */
    public boolean testConnection() {
        try {
            // Test if we can connect and get collection names
            List<String> collectionNames = mongoTemplate.getCollectionNames().stream().toList();
            log.info("Successfully connected to MongoDB. Available collections: {}", collectionNames);
            
            // Check if the merchLookup collection exists
            boolean merchLookupExists = collectionNames.contains("merchLookup");
            log.info("merchLookup collection exists: {}", merchLookupExists);
            
            if (!merchLookupExists) {
                log.warn("merchLookup collection not found in MongoDB!");
                return false;
            }
            
            // Test if we can retrieve data
            long count = merchLookupRepository.count();
            log.info("Total records in merchLookup: {}", count);
            
            if (count == 0) {
                log.warn("No records found in merchLookup collection!");
                return false;
            }
            
            // Get a sample record
            List<MerchLookup> sampleRecords = merchLookupRepository.findAll().stream().limit(1).toList();
            
            if (sampleRecords.isEmpty()) {
                log.warn("Failed to retrieve sample record!");
                return false;
            }
            
            MerchLookup sample = sampleRecords.get(0);
            log.info("Successfully retrieved a sample record:");
            log.info("  ID: {}", sample.getId());
            log.info("  BatchCd: {}", sample.getBatchCd());
            log.info("  MerchNo: {}", sample.getMerchNo());
            log.info("  CreatedBy: {}", sample.getAudit().getCreatedBy());
            
            return true;
            
        } catch (Exception e) {
            log.error("Error connecting to MongoDB: {}", e.getMessage(), e);
            return false;
        }
    }
}

@Autowired
private MongoConnectionTest mongoConnectionTest;

@Override
public void execute(JobEvent event) {
    log.info("Executing Merchant Number job...");
    
    // Test MongoDB connection first
    boolean mongoConnected = mongoConnectionTest.testConnection();
    if (!mongoConnected) {
        log.error("Failed to connect to MongoDB or retrieve data. Aborting job.");
        return;
    }
    
    // Continue with the rest of your job...
}

@RestController
@RequestMapping("/api/test")
public class TestController {
    
    @Autowired
    private MongoConnectionTest mongoConnectionTest;
    
    @GetMapping("/mongo")
    public ResponseEntity<String> testMongoConnection() {
        boolean connected = mongoConnectionTest.testConnection();
        
        if (connected) {
            return ResponseEntity.ok("MongoDB connection successful! Data retrieved.");
        } else {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("MongoDB connection failed or no data found. Check logs for details.");
        }
    }
}
