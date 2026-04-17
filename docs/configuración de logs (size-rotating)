$file = "C:\wildfly-39.0.1.Final\standalone\configuration\standalone.xml"

$old = '            <periodic-rotating-file-handler name="FILE" autoflush="true">
                <formatter>
                    <named-formatter name="PATTERN"/>
                </formatter>
                <file relative-to="jboss.server.log.dir" path="server.log"/>
                <suffix value=".yyyy-MM-dd"/>
                <append value="true"/>
            </periodic-rotating-file-handler>'

$new = '            <size-rotating-file-handler name="FILE" autoflush="true" rotate-on-boot="true">
                <formatter>
                    <named-formatter name="PATTERN"/>
                </formatter>
                <file relative-to="jboss.server.log.dir" path="server.log"/>
                <rotate-size value="50m"/>
                <max-backup-index value="5"/>
                <append value="true"/>
            </size-rotating-file-handler>'

(Get-Content $file -Raw).Replace($old, $new) | Set-Content $file -NoNewline
