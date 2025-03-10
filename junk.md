package com.wellsfargo.ccds.batchfacade.integration.mongo.repositories;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookup;

@Repository
public interface MerchLookupRepository extends MongoRepository<MerchLookup, String> {
    
    // Find all records with pagination
    Page<MerchLookup> findAll(Pageable pageable);
    
    // Find by specific fields if needed
    Page<MerchLookup> findByBatchCd(String batchCd, Pageable pageable);
    
    Page<MerchLookup> findByClient(String client, Pageable pageable);
    
    // Find records created after a certain date
    Page<MerchLookup> findByAudit_CreatedTsAfter(LocalDateTime timestamp, Pageable pageable);
}
