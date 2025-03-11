package com.wellsfargo.ccds.batchfacade.integration.service;

import java.util.ArrayList;
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
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


package com.wellsfargo.ccds.batchfacade.integration.service;

import java.util.ArrayList;
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;

import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookupRap;
import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.SuperCheckLookup;
import com.wellsfargo.ccds.batchfacade.integration.mongo.repositories.MerchLookupRapRepository;
import com.wellsfargo.ccds.batchfacade.integration.mongo.repositories.SuperCheckLookupRepository;

import lombok.extern.slf4j.Slf4j;

@Service
@Slf4j
public class RapSyncService {

    @Autowired
    private MerchLookupRapRepository merchLookupRapRepository;
    
    @Autowired
    private SuperCheckLookupRepository superCheckLookupRepository;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Value("${batch.rapsync.page.size:10}")
    private int pageSize;
    
    /**
     * Returns a list of all pages of MerchLookupRap data
     * 
     * @return List of pages containing MerchLookupRap data
     */
    public List<Page<MerchLookupRap>> getAllMerchLookupRapPages() {
        log.info("Retrieving all pages of merchant lookup RAP data with page size: {}", pageSize);
        
        List<Page<MerchLookupRap>> allPages = new ArrayList<>();
        int pageNumber = 0;
        boolean hasMoreData = true;
        
        while (hasMoreData) {
            Pageable pageable = PageRequest.of(pageNumber, pageSize);
            Page<MerchLookupRap> currentPage = merchLookupRapRepository.findAll(pageable);
            
            if (currentPage.hasContent()) {
                allPages.add(currentPage);
                pageNumber++;
            } else {
                hasMoreData = false;
            }
        }
        
        log.info("Retrieved {} pages of merchant lookup RAP data", allPages.size());
        return allPages;
    }
    
    /**
     * Returns a list of all pages of SuperCheckLookup data
     * 
     * @return List of pages containing SuperCheckLookup data
     */
    public List<Page<SuperCheckLookup>> getAllSuperCheckLookupPages() {
        log.info("Retrieving all pages of super check lookup data with page size: {}", pageSize);
        
        List<Page<SuperCheckLookup>> allPages = new ArrayList<>();
        int pageNumber = 0;
        boolean hasMoreData = true;
        
        while (hasMoreData) {
            Pageable pageable = PageRequest.of(pageNumber, pageSize);
            Page<SuperCheckLookup> currentPage = superCheckLookupRepository.findAll(pageable);
            
            if (currentPage.hasContent()) {
                allPages.add(currentPage);
                pageNumber++;
            } else {
                hasMoreData = false;
            }
        }
        
        log.info("Retrieved {} pages of super check lookup data", allPages.size());
        return allPages;
    }
    
    /**
     * Calculates the total number of MerchLookupRap pages
     * 
     * @return Total number of MerchLookupRap pages
     */
    public int getTotalMerchLookupRapPages() {
        long totalCount = merchLookupRapRepository.count();
        int totalPages = (int) Math.ceil((double) totalCount / pageSize);
        
        log.info("Total merchant lookup RAP records: {}, Total pages: {}", totalCount, totalPages);
        return totalPages;
    }
    
    /**
     * Calculates the total number of SuperCheckLookup pages
     * 
     * @return Total number of SuperCheckLookup pages
     */
    public int getTotalSuperCheckLookupPages() {
        long totalCount = superCheckLookupRepository.count();
        int totalPages = (int) Math.ceil((double) totalCount / pageSize);
        
        log.info("Total super check lookup records: {}, Total pages: {}", totalCount, totalPages);
        return totalPages;
    }
}

