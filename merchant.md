package com.wellsfargo.ccds.batchfacade.jobs.impl;

import com.wellsfargo.ccds.batchfacade.integration.message.event.JobEvent;
import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookupRap;
import com.wellsfargo.ccds.batchfacade.integration.mongo.repositories.MerchLookupRapRepository;
import com.wellsfargo.ccds.batchfacade.jobs.JobStrategy;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class MerchantNumberJobStrategy implements JobStrategy {

    @Autowired
    private MerchLookupRapRepository merchLookupRapRepository;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;

    private static final int PAGE_SIZE = 500; // Adjust batch size based on performance testing

    @Override
    public void execute(JobEvent event) {
        System.out.println("Executing Merchant Number job...");
        
        int page = 0;
        Page<MerchLookupRap> pagedData;
        
        do {
            pagedData = merchLookupRapRepository.findAll(PageRequest.of(page, PAGE_SIZE));
            List<MerchLookupRap> merchList = pagedData.getContent();
            
            if (!merchList.isEmpty()) {
                batchInsertToSQL(merchList);
            }
            page++;
        } while (pagedData.hasNext());
        
        runStoredProcedure();
    }

    private void batchInsertToSQL(List<MerchLookupRap> merchList) {
        String sql = "INSERT INTO MERCHANT_NUMBER_LOOKUP_BATCH (Batch_Code, Tran_Code, Client, System, Prin, Merchant) VALUES (?, ?, ?, ?, ?, ?)";
        
        jdbcTemplate.batchUpdate(sql, merchList, merchList.size(), (ps, merch) -> {
            ps.setString(1, merch.getBatchCd());
            ps.setString(2, merch.getTranCd());
            ps.setString(3, merch.getClient());
            ps.setString(4, merch.getSystem());
            ps.setString(5, "PrinValue"); // Placeholder if needed
            ps.setString(6, merch.getMerchNO());
        });
    }

    private void runStoredProcedure() {
        System.out.println("Running stored procedure to load data into materialized view...");
        jdbcTemplate.execute("CALL LoadMaterializedViewProcedure()");
    }
}



package com.wellsfargo.ccds.batchfacade.jobs.impl;

import com.wellsfargo.ccds.batchfacade.integration.message.event.JobEvent;
import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookupRap;
import com.wellsfargo.ccds.batchfacade.integration.mongo.repositories.MerchLookupRapRepository;
import com.wellsfargo.ccds.batchfacade.jobs.JobStrategy;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;
import javax.transaction.Transactional;
import java.util.List;

@Component
public class MerchantNumberJobStrategy implements JobStrategy {

    @Autowired
    private MerchLookupRapRepository merchLookupRapRepository;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    private static final int PAGE_SIZE = 100; // Efficient pagination size

    @Override
    @Transactional
    public void execute(JobEvent event) {
        System.out.println("Executing Merchant Number job...");

        int pageNumber = 0;
        Page<MerchLookupRap> page;

        do {
            page = merchLookupRapRepository.findAll(PageRequest.of(pageNumber, PAGE_SIZE));
            List<MerchLookupRap> merchLookupRaps = page.getContent();

            if (!merchLookupRaps.isEmpty()) {
                batchInsertToSql(merchLookupRaps);
            }
            pageNumber++;
        } while (page.hasNext());

        runStoredProcedure();
    }

    private void batchInsertToSql(List<MerchLookupRap> merchLookupRaps) {
        String sql = "INSERT INTO ${dbschema}.MERCHANT_NUMBER_LOOKUP_BATCH " +
                "(Batch_Code, Tran_Code, Client, System, Merchant) VALUES (?, ?, ?, ?, ?)";

        jdbcTemplate.batchUpdate(sql, merchLookupRaps, PAGE_SIZE, (ps, merch) -> {
            ps.setString(1, merch.getBatchCd());
            ps.setString(2, merch.getTranCd());
            ps.setString(3, merch.getClient());
            ps.setString(4, merch.getSystem());
            ps.setString(5, merch.getMerchNO());
        });
    }

    private void runStoredProcedure() {
        String sql = "CALL Load_MerchantNumber_MaterializedView()"; // Adjust with actual procedure name
        jdbcTemplate.execute(sql);
    }
}



package com.wellsfargo.ccds.batchfacade.integration.mongo.entities;

import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Data
@Document(collection = "MerchLookupRap")
public class MerchLookupRap {

    @Id
    private String id;
    private String batchCd;
    private String tranCd;
    private String client;
    private String system;
    private String merchNO;

     @Override
    public String toString() {
        return "MerchLookupRap{" +
                "id='" + id + '\'' +
                ", batchCd='" + batchCd + '\'' +
                ", tranCd='" + tranCd + '\'' +
                ", client='" + client + '\'' +
                ", system='" + system + '\'' +
                ", merchNO='" + merchNO + '\'' +
                '}';
    }

}

package com.wellsfargo.ccds.batchfacade.integration.mongo.repositories;

import com.wellsfargo.ccds.batchfacade.integration.mongo.entities.MerchLookupRap;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface MerchLookupRapRepository extends MongoRepository<MerchLookupRap, String> {
}

private void batchInsertToSQL(List<MerchLookupRap> merchList) {
    String sql = "INSERT INTO MERCHANT_NUMBER_LOOKUP_BATCH (Batch_Code, Tran_Code, Client, System, Prin, Merchant) VALUES (?, ?, ?, ?, ?, ?)";

    jdbcTemplate.batchUpdate(sql, merchList, merchList.size(), (ps, merch) -> {
        ps.setString(1, merch.getBatchCd());
        ps.setString(2, merch.getTranCd());
        ps.setString(3, merch.getClient());
        ps.setString(4, merch.getSystem());
        ps.setString(5, "PrinValue"); // Placeholder if needed
        ps.setString(6, merch.getMerchNO());
    });
}

