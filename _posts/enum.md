
import lombok.Data;

@Data
public class TKeyInfo {

    /**
     * 订单类型
     */
    @Deprecated
    private TBizType bizType;

    /**
     * 订单类型
     */
    @Deprecated
    private TGoodsType goodsType;

    /**
     * 订单类型
     */
    @Deprecated
    private TUmpType umpType;

    /**
     * 订单类型
     */
    @Deprecated
    private TDeliveryType deliveryType;

    /**
     * 订单类型
     */
    @Deprecated
    private TPayType payType;
    
    /**
     * 构造
     */
    public TKeyInfo(TBizType bizType,
                    TGoodsType goodsType,
                    TPayType payType,
                    TDeliveryType deliveryType,
                    TUmpType umpType
                    ) {
        this.goodsType = goodsType;
        this.bizType = bizType;
        this.deliveryType = deliveryType;
        this.umpType = umpType;
        this.payType = payType;
    }

    public TKeyInfo(TBizType bizType,
                    TGoodsType goodsType,
                    TPayType payType,
                    TDeliveryType deliveryType,
                    TUmpType umpType,
                    Long tags
    ) {
        this.goodsType = goodsType;
        this.bizType = bizType;
        this.deliveryType = deliveryType;
        this.umpType = umpType;
        this.payType = payType;
        this.tags = tags;
    }

    /**
     * 添加tag , 对于五维度依然不能确定的流程 , 需要添加tag标
     * @param values 多个tag位
     * @return 当前对象 测试这边没有多tag叠加的场景，弃用吧
     */
    @Deprecated
    public TKeyInfo addTags(TTagEnum... values){
        for (TTagEnum value : values) {
            this.tags |= (1L << value.index());
        }
        return this;
    }

    /**
     * 添加tag , 对于五维度依然不能确定的流程 , 需要添加tag标
     * @return 当前对象
     */
    public TKeyInfo addTags(TTagEnum tag){
        this.tags = tag.index();
        return this;
    }

    @Override
    public String toString(){
        String bizTypeString = "|bizType:"+bizType.getDesc();
        String goodsTypeString = "|goodsType:"+goodsType.getDesc();
        String deliveryTypeString = "|deliveryType:"+deliveryType.getDesc();
        String umpTypeString = "|umpType:"+umpType.getDesc();
        String payTypeString = "|payType:"+payType.getDesc();
        return bizTypeString+goodsTypeString+deliveryTypeString+umpTypeString+payTypeString;
    }
}
