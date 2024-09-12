.PHONY: all
all: suit-manifest-multiple.cddl

suit-manifest-multiple.cddl: draft-ietf-suit-manifest.cddl rfc-9052.cddl
	cat $^ ../draft-ietf-suit-trust-domains.cddl > $@

draft-ietf-suit-manifest.cddl:
	curl --retry 3 https://raw.githubusercontent.com/suit-wg/manifest-spec/master/draft-ietf-suit-manifest.cddl -o $@

rfc-9052.cddl: rfc-9052.xml
	sed -n -e '/<sourcecode type="CDDL"/,/<\/sourcecode/ p' $< | sed -e '/<sourcecode type="CDDL"/ d' -e '/<\/sourcecode/ d' -e '/<\/section>/,/<figure title=""/ d' -e 's/\&gt;/>/g' > $@

rfc-9052.xml:
	curl --retry 3 https://raw.githubusercontent.com/cose-wg/cose-rfc8152bis/master/draft-ietf-cose-rfc8152bis-struct.xml -o $@


../examples/example1_process.suit:
../examples/example2_integrated.suit:
	$(MAKE) -C ../examples

.PHONY: validate
validate: suit-manifest-multiple.cddl ../examples/example1_process.suit ../examples/example2_integrated.suit
	RUBYOPT="-W0" cddl $< validate ../examples/example1_process.suit
	@echo "[SUCCESS] example1_process matches the CDDL"
	RUBYOPT="-W0" cddl $< validate ../examples/example2_integrated.suit
	@echo "[SUCCESS] example2_integrated matches the CDDL"

.PHONY: clean
clean:
	$(RM) *.cddl *.xml
