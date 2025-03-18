1. Add Diagnostic Logging in the MongoDB Service
public Page<MerchLookup> getPage(int pageNumber) {
    log.info("Retrieving page {} of merchant lookup data", pageNumber);
    Pageable pageable = PageRequest.of(pageNumber, pageSize);
    Page<MerchLookup> page = merchLookupRepository.findAll(pageable);
    
    // Diagnostic check - log record count and sample data
    if (page.hasContent()) {
        log.info("Retrieved {} records from MongoDB", page.getNumberOfElements());
        log.debug("Sample record from MongoDB: {}", page.getContent().get(0));
    } else {
        log.warn("No data found in MongoDB for page {}", pageNumber);
    }
    
    return page;
}


2. Create a Test Endpoint for Direct Verification:
@RestController
@RequestMapping("/api/test")
public class MongoDbTestController {
    
    @Autowired
    private MerchLookupMongoService merchLookupMongoService;
    
    @GetMapping("/mongo/check")
    public ResponseEntity<Map<String, Object>> checkMongoDbData() {
        Map<String, Object> response = new HashMap<>();
        
        try {
            int totalPages = merchLookupMongoService.getTotalPages();
            long totalRecords = merchLookupMongoService.getTotalRecordCount();
            
            response.put("status", "success");
            response.put("totalPages", totalPages);
            response.put("totalRecords", totalRecords);
            
            if (totalRecords > 0) {
                // Get sample data from first page
                Page<MerchLookup> firstPage = merchLookupMongoService.getPage(0);
                response.put("sampleData", firstPage.getContent().subList(0, 
                    Math.min(3, firstPage.getContent().size())));
            }
            
            return ResponseEntity.ok(response);
        } catch (Exception e) {
            response.put("status", "error");
            response.put("message", e.getMessage());
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
        }
    }
}

1.Add Validation in SQL Service
@Transactional
public int saveBatch(List<MerchLookup> merchLookups) {
    log.info("Saving batch of {} merchant lookup records to SQL database", merchLookups.size());
    
    List<MerchantNumberLookup> sqlEntities = merchLookups.stream()
            .map(this::convertToSqlEntity)
            .collect(Collectors.toList());
    
    List<MerchantNumberLookup> savedEntities = merchantNumberLookupRepository.saveAll(sqlEntities);
    
    // Validate insertion
    if (savedEntities.size() != merchLookups.size()) {
        log.warn("Not all records were saved! Expected: {}, Actual: {}", 
            merchLookups.size(), savedEntities.size());
    }
    
    // You could add a more detailed check by querying one or more of the just-inserted records
    if (!savedEntities.isEmpty()) {
        MerchantNumberLookup firstSaved = savedEntities.get(0);
        boolean exists = merchantNumberLookupRepository.existsById(firstSaved.getId());
        log.info("Verified sample record exists in database: {}", exists);
    }
    
    return savedEntities.size();
}

2. Create a SQL Validation Method
public boolean validateSqlInsertion(List<MerchLookup> originalRecords, List<MerchantNumberLookup> savedRecords) {
    // Check record counts
    if (originalRecords.size() != savedRecords.size()) {
        log.error("Record count mismatch after insertion. Original: {}, Saved: {}", 
            originalRecords.size(), savedRecords.size());
        return false;
    }
    
    // Sample check on a few records
    for (int i = 0; i < Math.min(3, originalRecords.size()); i++) {
        MerchLookup original = originalRecords.get(i);
        MerchantNumberLookup saved = savedRecords.get(i);
        
        if (!original.getMerchNo().equals(saved.getMerchNo()) || 
            !original.getBatchCd().equals(saved.getBatchCd())) {
            log.error("Data mismatch for record {}: MongoDB: {}, SQL: {}", 
                i, original, saved);
            return false;
        }
    }
    
    return true;
}

3. Add Direct SQL Query for Verification
public int countRecordsInSqlTable() {
    Integer count = jdbcTemplate.queryForObject(
        "SELECT COUNT(*) FROM MERCHANT_NUMBER_LOOKUP_BATCH", Integer.class);
    log.info("Current record count in SQL table: {}", count);
    return count != null ? count : 0;
}

4. Test Endpoint for SQL Verification:
@RestController
@RequestMapping("/api/test")
public class SqlDbTestController {
    
    @Autowired
    private MerchantNumberSqlService merchantNumberSqlService;
    
    @GetMapping("/sql/check")
    public ResponseEntity<Map<String, Object>> checkSqlDbData() {
        Map<String, Object> response = new HashMap<>();
        
        try {
            int recordCount = merchantNumberSqlService.countRecordsInSqlTable();
            
            response.put("status", "success");
            response.put("recordCount", recordCount);
            
            // You could add a sample query here to get a few records
            
            return ResponseEntity.ok(response);
        } catch (Exception e) {
            response.put("status", "error");
            response.put("message", e.getMessage());
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
        }
    }
}


5. Integration Testing the Entire Flow
You could also create a simple test job that executes a single page transfer:

@Service
public class BatchJobTestService {
    
    @Autowired
    private MerchLookupMongoService merchLookupMongoService;
    
    @Autowired
    private MerchantNumberSqlService merchantNumberSqlService;
    
    public Map<String, Object> testSinglePageTransfer() {
        Map<String, Object> result = new HashMap<>();
        
        try {
            // Record counts before
            int sqlCountBefore = merchantNumberSqlService.countRecordsInSqlTable();
            result.put("sqlCountBefore", sqlCountBefore);
            
            // Get first page from MongoDB
            Page<MerchLookup> firstPage = merchLookupMongoService.getPage(0);
            result.put("mongoPageSize", firstPage.getSize());
            result.put("mongoRecordsFound", firstPage.getNumberOfElements());
            
            if (firstPage.hasContent()) {
                // Insert to SQL
                int inserted = merchantNumberSqlService.saveBatch(firstPage.getContent());
                result.put("recordsInserted", inserted);
                
                // Count after
                int sqlCountAfter = merchantNumberSqlService.countRecordsInSqlTable();
                result.put("sqlCountAfter", sqlCountAfter);
                result.put("difference", sqlCountAfter - sqlCountBefore);
                
                result.put("status", "success");
            } else {
                result.put("status", "warning");
                result.put("message", "No data found in MongoDB to transfer");
            }
        } catch (Exception e) {
            result.put("status", "error");
            result.put("message", e.getMessage());
        }
        
        return result;
    }
}
