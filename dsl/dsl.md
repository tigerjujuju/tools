```java

@Configuration
@RequiredArgsConstructor
public class SampleFlowConfig implements FlowRegistrar {
 
    // 실제 서비스 빈 주입 예시
    // private final UserService userService;
    // private final CartService cartService;
    // private final ExternalOrderService externalOrderService;
 
    @Override
    public void configureFlows(FlowRegistry registry) {
 
        // ── S0001 / A01: Primary + Secondary 조회 혼합 예시 ──
        registry.register(
                ActionKey.of("S0001", "A01"),
                FlowDefinition.builder()
                        // Task 1: Primary DB 사용자 조회 (readOnly)
                        .task(ctx -> {
                            // ctx.put("user", userService.findById(ctx.getLongParam("userId")));
                        }, TxOption.readOnly())
 
                        .forward("user/detail")
                        .build()
        );
 
        // ── S0001 / A02: Primary DB 트랜잭션 처리 예시 ──
        registry.register(
                ActionKey.of("S0001", "A02"),
                FlowDefinition.builder()
                        // Task 1: Primary DB 조회 후 컨텍스트에 보관
                        .task(ctx -> {
                            // ctx.put("user", userService.findById(ctx.getLongParam("userId")));
                        }, TxOption.readOnly())
 
                        // Task 2: Primary DB 쓰기 (별도 트랜잭션)
                        .task(ctx -> {
                            // cartService.addItem(ctx.get("user"), ctx.getLongParam("itemId"));
                        }, TxOption.defaults())
 
                        // Task 3: 트랜잭션 없이 외부 시스템 알림
                        .task(ctx -> {
                            // notificationService.sendPushAlert(ctx.get("user"));
                        }, TxOption.noTransaction())
 
                        .forward("cart/result")
                        .build()
        );
    }
}





    @SuppressWarnings("unchecked")
    public HashMap<String, Object> siaiAuditCtrctSave(WebmemberCtrctInfo ctrctInfo) {
        BrokerContext ctx = BrokerContext.init();
        try {
            ctx.put("WEBMEMBERCTRCTINFO", ctrctInfo);
 
            BrokerPipeline.create()
                    .call(new SiaiIServiceModule(), m -> m.siaiAuditCtrctSave(
                            ctx.require("WEBMEMBERCTRCTINFO", WebmemberCtrctInfo.class) /* allowedNull=false */
                    ))
                    .call(new AuditIServiceModule(), m -> m.auditRecord(
                            ctx.get("RETURNMAP", HashMap.class) /* allowedNull=true */
                    ))
                    .execute();
 
            return ctx.get("RETURNMAP", HashMap.class);
        } finally {
            BrokerContext.clear();
        }
    }
}


```

