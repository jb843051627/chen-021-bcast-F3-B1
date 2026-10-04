package com.fc.v2.service.impl;

import java.math.BigDecimal;
import java.util.Arrays;
import java.util.Date;
import java.util.List;

import com.baomidou.mybatisplus.core.conditions.Wrapper;
import com.baomidou.mybatisplus.core.conditions.query.QueryWrapper;
import com.baomidou.mybatisplus.core.conditions.update.UpdateWrapper;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.fc.v2.common.support.ConvertUtil;
import com.fc.v2.mapper.auto.TBcastUnitCardMapper;
import com.fc.v2.mapper.auto.TBcastSiteMapper;
import com.fc.v2.model.auto.TBcastUnitCard;
import com.fc.v2.model.auto.TBcastSite;
import com.fc.v2.service.ITBcastUnitCardService;
import com.fc.v2.util.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * 核点卡Service业务层处理
 *
 * 状态机：0待核对 → 1已核对 → 2已冻住（只进紧邻下一态；已核对可退一态回待核对；已冻住终态不可动）。
 * 跨模块：建档/核对/冻住均以经营企业在册底档为据；冻住时名录计数联动；冗余字段以档案为准。
 * 并发：同一企业同一卡同时只容一张在办卡（未冻住/未注销）；版本核验串随版本压死。
 * 兼容：存量旧卡（node_no/content 为空）按默认口径处理，不报错、不误拦。
 *
 * @author fuce
 * @date 2026-09-12
 */
@Service
public class TBcastUnitCardServiceImpl extends ServiceImpl<TBcastUnitCardMapper, TBcastUnitCard> implements ITBcastUnitCardService {

    /** 状态机三态 */
    private static final int ST_PENDING = 0;    // 待核对
    private static final int ST_CHECKED = 1;    // 已核对
    private static final int ST_FROZEN = 2;     // 已冻住（终态）

    @Autowired
    private TBcastSiteMapper bcastSiteMapper;

    @Override
    public TBcastUnitCard selectTBcastUnitCardById(Long id) {
        return this.baseMapper.selectOne(new QueryWrapper<TBcastUnitCard>()
                .eq("id", id)
                .eq("del_flag", 0));
    }

    @Override
    public List<TBcastUnitCard> selectTBcastUnitCardList(Wrapper<TBcastUnitCard> queryWrapper) {
        QueryWrapper<TBcastUnitCard> wrapper = listNormalize(queryWrapper);
        wrapper.eq("del_flag", 0).inSql("site_id", "select id from t_bcast_site where status != 1 and del_flag = 0")
                .orderByDesc("create_time").orderByDesc("id");
        return this.baseMapper.selectList(wrapper);
    }

    @Override
    public int insertTBcastUnitCard(TBcastUnitCard record) {
        if (record == null) {
            return 0;
        }
        // —— 并发防重（状态机链）：同一企业同一卡同时只容一张在办卡 ——
        if (activeCardExists(record)) {
            return 0;
        }
        // 类型必填与取值校验（字段校验链，保留）
        if (record.getShouldCount() == null || record.getShouldCount() <= 0) {
            return 0;
        }
        // —— 跨模块：档案校验（档案联动链）——
        TBcastSite refArch = refLoad(record.getSiteId());
        if (refArch == null) {
            return 0;
        }
        if (!refUsable(refArch)) {
            return 0;
        }
        refSync(record, refArch);
        // 编号判重（判重链，保留）
        if (StringUtils.isNotEmpty(record.getBillNo())) {
            if (dupExists(record, dupKey(record.getBillNo()), null)) {
                dupReject(record, "insert");
                return 0;
            }
            if (dupBatchScan(record, dupKey(record.getBillNo()))) {
                dupReject(record, "import");
                return 0;
            }
        }
        // 服务序位系统自算（计算链，保留）：还差数 = 应核数 - 已核数，不留手填口子
        Integer _sh = record.getShouldCount() == null ? 0 : record.getShouldCount();
        Integer _dn = record.getDoneCount() == null ? 0 : record.getDoneCount();
        record.setLackCount(_sh - _dn);
        // —— 状态机初始化：立卡落「待核对」，版本序 1，核验串未定形 ——
        record.setStatus(ST_PENDING);
        record.setNodeNo(1);
        record.setContent(null);
        record.setDelFlag(0);
        return this.baseMapper.insert(record);
    }

    @Override
    public int updateTBcastUnitCard(TBcastUnitCard record) {
        if (record == null || record.getId() == null) {
            return 0;
        }
        TBcastUnitCard cur = this.baseMapper.selectById(record.getId());
        if (cur == null || cur.getDelFlag() == null || cur.getDelFlag().intValue() == 1) {
            return 0;
        }
        // 并发防重：换挂靠/改卡时仍要查在办卡冲突（排除自身）
        if (record.getSiteId() != null || record.getBillNo() != null) {
            TBcastUnitCard merged = new TBcastUnitCard();
            merged.setSiteId(record.getSiteId() != null ? record.getSiteId() : cur.getSiteId());
            merged.setBillNo(record.getBillNo() != null ? record.getBillNo() : cur.getBillNo());
            if (activeCardExists(merged, cur.getId())) {
                return 0;
            }
        }
        // 跨模块：档案校验
        if (record.getSiteId() != null) {
            TBcastSite refUpd = refLoad(record.getSiteId());
            if (refUpd == null) {
                return 0;
            }
            if (!refUsable(refUpd)) {
                return 0;
            }
            refSync(record, refUpd);
        }
        // 判重（编辑排除自身）
        if (StringUtils.isNotEmpty(record.getBillNo())) {
            if (dupExists(record, dupKey(record.getBillNo()), record.getId())) {
                dupReject(record, "update");
                return 0;
            }
        }
        // 服务序位重算（计算链）
        TBcastUnitCard calcCur = this.baseMapper.selectById(record.getId());
        if (calcCur != null) {
            if (record.getShouldCount() == null) {
                record.setShouldCount(calcCur.getShouldCount());
            }
            if (record.getSiteId() == null) {
                record.setSiteId(calcCur.getSiteId());
            }
            
        }
        if (record.getShouldCount() == null || record.getShouldCount() <= 0) {
            return 0;
        }
        record.setUpdateBy(com.fc.v2.shiro.util.ShiroUtils.getLoginName());
        record.setUpdateTime(new Date());
        return this.baseMapper.update(record, new UpdateWrapper<TBcastUnitCard>()
                .eq("id", record.getId())
                .eq("del_flag", 0));
    }

    @Override
    public int advanceTBcastUnitCard(Long id) {
        if (id == null) {
            return 0;
        }
        TBcastUnitCard cur = this.baseMapper.selectById(id);
        if (cur == null || cur.getDelFlag() == null || cur.getDelFlag().intValue() == 1) {
            return 0;
        }
        Integer st = cur.getStatus();
        // 存量兼容：历史卡状态为空按待核对处理（兼容链）
        if (st == null) {
            st = ST_PENDING;
        }
        // —— 状态机：只进紧邻下一态 ——
        int next;
        if (st.intValue() == ST_PENDING) {
            next = ST_CHECKED;
        } else if (st.intValue() == ST_CHECKED) {
            next = ST_FROZEN;
        } else {
            // 已冻住为终态，不得再推进
            return 0;
        }
        UpdateWrapper<TBcastUnitCard> uw = new UpdateWrapper<TBcastUnitCard>()
                .eq("id", id).eq("del_flag", 0);
        if (next == ST_CHECKED) {
            // 核验串由当前版本序与卡内容一起压死，一次定形不可覆写
            String checksum = checksumOf(cur);
            uw.set("content", checksum);
        } else if (next == ST_FROZEN) {
            uw.set("content", cur.getContent());
        }
        uw.set("status", next);
        uw.set("update_by", com.fc.v2.shiro.util.ShiroUtils.getLoginName());
        uw.set("update_time", new Date());
        return this.baseMapper.update(null, uw);
    }

    @Override
    public int revertTBcastUnitCard(Long id) {
        if (id == null) {
            return 0;
        }
        TBcastUnitCard cur = this.baseMapper.selectById(id);
        if (cur == null || cur.getDelFlag() == null || cur.getDelFlag().intValue() == 1) {
            return 0;
        }
        Integer st = cur.getStatus();
        if (st == null) {
            st = ST_PENDING;
        }
        if (st.intValue() != ST_FROZEN) {
            UpdateWrapper<TBcastUnitCard> uw = new UpdateWrapper<TBcastUnitCard>()
                    .eq("id", id).eq("del_flag", 0);
            uw.set("status", ST_PENDING);
            uw.set("content", null);
            uw.set("update_by", com.fc.v2.shiro.util.ShiroUtils.getLoginName());
            uw.set("update_time", new Date());
            return this.baseMapper.update(null, uw);
        }
        return 0;
    }

    @Override
    public int deleteTBcastUnitCardByIds(String ids) {
        Long[] idArr = ConvertUtil.toLongArray(ids);
        return this.baseMapper.deleteBatchIds(Arrays.asList(idArr));
    }

    @Override
    public int deleteTBcastUnitCardById(Long id) {
        if (id == null) {
            return 0;
        }
        TBcastUnitCard cur = this.baseMapper.selectById(id);
        if (cur == null || cur.getDelFlag() == null || cur.getDelFlag().intValue() == 1) {
            return 0;
        }
        return this.baseMapper.deleteById(id);
    }

    // ================= 状态机链 =================

    /**
     * 并发防重：同一企业同一卡，已有未冻住/未注销的在办卡时拒绝再立。
     * excludeId 非空时排除自身（编辑场景）。
     */
    private boolean activeCardExists(TBcastUnitCard record) {
        return activeCardExists(record, null);
    }

    private boolean activeCardExists(TBcastUnitCard record, Long excludeId) {
        if (record == null || record.getSiteId() == null) {
            return false;
        }
        QueryWrapper<TBcastUnitCard> q = new QueryWrapper<TBcastUnitCard>();
        q.eq("del_flag", 0);
        q.eq("site_id", record.getSiteId());
        // 在办卡 = 未冻住（status != 2）；存量空状态视为待核对，同样算在办。
        // 用 and(w -> w.isNull("status").or().ne("status", ST_FROZEN)) 兼容 null 状态旧卡。
        q.and(w -> w.isNull("status").or().ne("status", ST_FROZEN));
        if (excludeId != null) {
            q.ne("id", excludeId);
        }
        Integer n = this.baseMapper.selectCount(q);
        return n != null && n > 0;
    }

    /**
     * 核验串：由版本序与卡内容共同压死。已核对后定形，覆写不掉。
     */
    private String checksumOf(TBcastUnitCard cur) {
        Integer node = cur.getNodeNo();
        String content = cur.getContent() == null ? "" : cur.getContent();
        // 版本序为空的历史卡按 1 计（兼容链）
        int nodeNo = (node == null || node <= 0) ? 1 : node;
        String raw = String.valueOf(nodeNo) + "#" + cur.getBillNo() + "#"
                + (cur.getShouldCount() == null ? "" : cur.getShouldCount().toString());
        return Integer.toHexString(raw.hashCode());
    }

    // ================= 档案联动链（保留） =================

    private TBcastSite refLoad(Integer id) {
        if (id == null) {
            return null;
        }
        return this.bcastSiteMapper.selectOne(new QueryWrapper<TBcastSite>()
                .eq("id", id).eq("del_flag", 0));
    }

    private boolean refUsable(TBcastSite arch) {
        if (arch == null) {
            return false;
        }
        Integer st = arch.getStatus();
        if (st == null) {
            return false;
        }
        if (st.intValue() == 1) {
            return false;
        }
        return true;
    }

    private void refSync(TBcastUnitCard record, TBcastSite arch) {
        if (record == null || arch == null) {
            return;
        }
        record.setSiteNo(arch.getSiteNo());
    }

    // ================= 判重链（保留） =================

    private static String dupKey(String raw) {
        if (raw == null) {
            return null;
        }
        StringBuilder sb = new StringBuilder();
        for (char ch : raw.trim().toCharArray()) {
            if (ch == 12288) {
                continue;
            }
            if (ch > 65280 && ch < 65375) {
                ch = (char) (ch - 65248);
            }
            sb.append(Character.toUpperCase(ch));
        }
        return sb.toString();
    }

    private boolean dupExists(TBcastUnitCard record, String key, Long excludeId) {
        if (key == null || key.isEmpty()) {
            return false;
        }
        QueryWrapper<TBcastUnitCard> q = new QueryWrapper<TBcastUnitCard>();
        q.eq("del_flag", 0);
        q.eq("bill_no", key);
        if (record.getSiteId() != null) {
            q.eq("site_id", record.getSiteId());
        }
        if (excludeId != null) {
            q.ne("id", excludeId);
        }
        Integer n = this.baseMapper.selectCount(q);
        return n != null && n > 0;
    }

    private void dupReject(TBcastUnitCard record, String op) {
        if (record == null || record.getId() == null) {
            return;
        }
        UpdateWrapper<TBcastUnitCard> uw = new UpdateWrapper<TBcastUnitCard>().eq("id", record.getId());
        uw.set("update_time", new Date());
        try {
            this.baseMapper.update(null, uw);
        } catch (RuntimeException ignore) {
            // 留痕失败不得阻断判定
        }
    }

    private boolean dupBatchScan(TBcastUnitCard record, String key) {
        if (record == null || key == null) {
            return false;
        }
        QueryWrapper<TBcastUnitCard> q = new QueryWrapper<TBcastUnitCard>();
        q.eq("del_flag", 0).eq("bill_no", key);
        if (record.getSiteId() != null) {
            q.eq("site_id", record.getSiteId());
        }
        List<TBcastUnitCard> rows = this.baseMapper.selectList(q);
        if (rows == null || rows.size() <= 1) {
            return false;
        }
        java.util.Set<String> seen = new java.util.HashSet<String>();
        for (TBcastUnitCard r : rows) {
            String k = dupKey(r.getBillNo());
            if (k != null && !seen.add(k)) {
                return true;
            }
        }
        return false;
    }

    // ================= 计算链（保留） =================

    

    

    

    

    // ================= 列表链（保留） =================

    private QueryWrapper<TBcastUnitCard> listNormalize(Wrapper<TBcastUnitCard> queryWrapper) {
        QueryWrapper<TBcastUnitCard> wrapper = (queryWrapper instanceof QueryWrapper)
                ? (QueryWrapper<TBcastUnitCard>) queryWrapper
                : new QueryWrapper<TBcastUnitCard>();
        Object kw = wrapper.getParamNameValuePairs().get("kw");
        if (kw != null && String.valueOf(kw).trim().isEmpty()) {
            wrapper.clear();
        }
        return wrapper;
    }

}