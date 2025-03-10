package com.wellsfargo.ccds.batchfacade.integration.service;

import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.time.LocalDateTime;
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.jdbc.core.BatchPreparedStatementSetter;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;

import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookup;
import com.wellsfargo.ccds.batchfacade.integration.mongo.repositories.MerchLookupRepository;

import lombok.extern.slf4j.Slf4j;

@Service
@Slf4j
public class MerchantNumberService {

    @Autowired
    private MerchLookupRepository merchLookupRepository;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    /**
     * Fetches a specific page of merchant lookup data
     * 
     * @param pageNumber The page number to retrieve (0-based)
     * @param pageSize The size of each page
     * @return Page of MerchLookup entities
     */
    public Page<MerchLookup> findByPageNumber(int pageNumber, int pageSize) {
        log.info("Fetching merchant lookup data for page: {} with size: {}", pageNumber, pageSize);
        Pageable pageable = PageRequest.of(pageNumber, pageSize);
        return merchLookupRepository.findAll(pageable);
    }
    
    /**
     * Finds merchant lookup data by batch code with pagination
     * 
     * @param batchCd The batch code to filter by
     * @param pageNumber The page number to retrieve (0-based)
     * @param pageSize The size of each page
     * @return Page of MerchLookup entities
     */
    public Page<MerchLookup> findByBatchCd(String batchCd, int pageNumber, int pageSize) {
        log.info("Fetching merchant lookup data for batchCd: {}, page: {} with size: {}", batchCd, pageNumber, pageSize);
        Pageable pageable = PageRequest.of(pageNumber, pageSize);
        return merchLookupRepository.findByBatchCd(batchCd, pageable);
    }
    
    /**
     * Finds merchant lookup data by client with pagination
     * 
     * @param client The client to filter by
     * @param pageNumber The page number to retrieve (0-based)
     * @param pageSize The size of each page
     * @return Page of MerchLookup entities
     */
    public Page<MerchLookup> findByClient(String client, int pageNumber, int pageSize) {
        log.info("Fetching merchant lookup data for client: {}, page: {} with size: {}", client, pageNumber, pageSize);
        Pageable pageable = PageRequest.of(pageNumber, pageSize);
        return merchLookupRepository.findByClient(client, pageable);
    }
    
    /**
     * Finds merchant lookup data created after a specified timestamp with pagination
     * 
     * @param timestamp The timestamp to filter by
     * @param pageNumber The page number to retrieve (0-based)
     * @param pageSize The size of each page
     * @return Page of MerchLookup entities
     */
    public Page<MerchLookup> findByCreatedTsAfter(LocalDateTime timestamp, int pageNumber, int pageSize) {
        log.info("Fetching merchant lookup data created after: {}, page: {} with size: {}", timestamp, pageNumber, pageSize);
        Pageable pageable = PageRequest.of(pageNumber, pageSize);
        return merchLookupRepository.findByAudit_CreatedTsAfter(timestamp, pageable);
    }
    
    /**
     * Fetches data from MongoDB in pages of specified size
     * 
     * @param pageSize The size of each page
     * @return Total number of records processed
     */
    public int processDataInPages(int pageSize) {
        log.info("Starting to process merchant lookup data in pages of size: {}", pageSize);
        
        int pageNumber = 0;
        int totalProcessed = 0;
        boolean hasMoreData = true;
        
        while (hasMoreData) {
            Page<MerchLookup> merchantPage = findByPageNumber(pageNumber, pageSize);
            
            List<MerchLookup> merchantData = merchantPage.getContent();
            
            if (merchantData.isEmpty()) {
                hasMoreData = false;
            } else {
                int processed = insertIntoSqlDatabase(merchantData);
                totalProcessed += processed;
                
                log.info("Processed page {}: {} records", pageNumber, processed);
                pageNumber++;
            }
        }
        
        log.info("Total processed records: {}", totalProcessed);
        return totalProcessed;
    }
    
    /**
     * Inserts the merchant data into SQL database
     * 
     * @param merchantData List of merchant lookup records
     * @return Number of records inserted
     */
    private int insertIntoSqlDatabase(List<MerchLookup> merchantData) {
        String sql = "INSERT INTO MERCHANT_NUMBER_LOOKUP_BATCH (BATCH_CD, TRAN_CD, CLIENT, MERCH_NO, CREATED_TS, CREATED_BY) " +
                    "VALUES (?, ?, ?, ?, ?, ?)";
        
        int[] updateCounts = jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                MerchLookup merchant = merchantData.get(i);
                ps.setString(1, merchant.getBatchCd());
                ps.setString(2, merchant.getTranCd());
                ps.setString(3, merchant.getClient());
                ps.setString(4, merchant.getMerchNo());
                ps.setObject(5, merchant.getAudit().getCreatedTs());
                ps.setString(6, merchant.getAudit().getCreatedBy());
            }
            
            @Override
            public int getBatchSize() {
                return merchantData.size();
            }
        });
        
        return updateCounts.length;
    }
    
    /**
     * Executes the stored procedure to refresh the materialized view
     */
    public void refreshMaterializedView() {
        log.info("Executing stored procedure to refresh materialized view");
        jdbcTemplate.execute("CALL REFRESH_MERCHANT_NUMBER_LOOKUP_MV()");
        log.info("Materialized view refresh completed");
    }
}
