## freewall sortable just open freewall file
```
/**
 * Created by yanshi0429 on 17/5/22.
 */
import * as template from './template.html';
import * as popTocMenuTemplate from '../toc/pop.toc.menu.html';
import popTocMenuCtrl from '../toc/pop.toc.menu.js';

class LayoutController {
    constructor($rootScope,
        portalService,
        $scope,
        $stateParams,
        $timeout,
        PortalConstant,
        $pbox,
        $filter) {
        this.$rootScope = $rootScope;
        this.portalService = portalService;
        this.$pbox = $pbox;
        this.$scope = $scope;
        this.$stateParams = $stateParams;
        this.$timeout = $timeout;
        this.constant = PortalConstant;
        this.$filter = $filter;
        this.panelId = $stateParams.id;
        this.panel = {};
        this.cellWidth = 150;
        this.cellHeight = 50 * 5 + 75;
        this.data = [];
    }
    //$ctrl.panel.privileges | hasPortalPrivilege:'editPanel'

    setCellHeight() {
        if (this.cellWidth < 250) {
            this.cellHeight = 40 * 5 + 75;
            this.isSmallWidget = true;
        } else {
            this.isSmallWidget = false;
            this.cellHeight = 50 * 5 + 75;
        }
    }

    getCellWidth() {
        this.cellWidth = parseInt((angular.element('#portal-layout').width() - 30) / 4);
        this.setCellHeight();

    }

    openSelectWidget() {
        this.portalService.openSelectWidget(this.panel);
    }

    getWidgets() {
        this.loadingDone = false;
        this.portalService.getPortalWidgets(this.panelId)
            .then((data) => {
                this.initLayout();
                this.loadingDone = true;
            }, (err) => {
                if (err.code == 1005) {
                    alert(1005)
                }
            });
    }

    getPanel() {
        this.panel = _.find(this.portalService.panels, {
            _id: this.panelId
        });
    }

    popTocMenu($event) {
        if (!this.$filter('hasPortalPanelOnePrivilege')(this.panel.privileges)) {
            return;
        }
        this.$pbox.open({
            event: $event,
            template: popTocMenuTemplate,
            controller: popTocMenuCtrl,
            resolve: {
                panel: () => {
                    return this.panel;
                }
            }
        });
    }

    initSize() {
        let self = this;
        let x = 0;
        let y = 0;
        _.each(self.portalService.widgets, function (n) {
            let _x = x + n.size.width;
            let _y = y + n.size.height;


            n.style = {
                width: n.size.width * self.cellWidth,
                height: n.size.height * self.cellHeight
            };
        });
    }

    setDataPosition() {
        let self = this;
        $('.widget-wrap').each(function () {
            let _po = $(this).position();
            let top = Math.ceil(_po.top / self.cellWidth);
            let left = Math.ceil(_po.left / self.cellHeight);
            let _widgetId = $(this).attr('widget-id');
            let _wid = _.find(self.portalService.widgets, {
                _id: _widgetId
            });
            if (_wid) {
                _wid.dataPosition = top + "-" + left;
            };
        });
    }

    initLayout() {
        let self = this;
        this.getCellWidth();
        this.initSize();
        this.$timeout(function () {
            if (self.wall) {
                self.wall = null;
            }
            self.wall = new Freewall("#portal-layout-container");
            self.wall.reset({
                draggable: self.$filter("hasPortalPrivilege")(self.panel.privileges, 'editPanel'),
                selector: '.widget-wrap',
                fixSize: 0,
                cellW: self.cellWidth,
                cellH: self.cellHeight,
                animate: true,
                cacheSize:true,
                gutterX: 10,
                gutterY: 10,
                onResize: function () {
                    self.wall.fitWidth();
                },
                onBlockDrop: function (e) {
                    self.setDataPosition();
                    self.portalService.widgets = _.sortBy(self.portalService.widgets, "dataPosition");
                    let _ids = _.map(self.portalService.widgets, function (item) {
                        return {
                            id: item._id,
                            name: item.title
                        };
                    });
                    self.portalService.updatePanelWidgetPosition(self.panel._id, _ids).then((data) => {}, (err) => {});
                },
                onComplete: function () {
                    self.setDataPosition();
                }
            });
            //self.wall.fitZone();
            self.wall.fitWidth();
            $(window).trigger("resize");
        });
    }

    $onInit() {
        this.getPanel();
        if (!this.panel) {
            this.loadingDone = true;
            return;
        }

        this.getWidgets();

        // $(window).resize(() => {
        //     this.layoutWidth = angular.element('#portal-layout').width();
        //     console.log(this.layoutWidth);
        // });

        let self = this;
        this.$scope.$on('refreshPanelWidgetLayout', () => {
            self.initLayout();
        });
    }
}

LayoutController.$inject = [
    '$rootScope',
    'PortalService',
    '$scope',
    '$stateParams',
    '$timeout',
    'PortalConstant',
    '$pbox',
    '$filter'
];

export default {
    selector: 'portalLayout',
    template: template,
    controller: LayoutController
};





<div class="module-header">
    <div class="title">
        <a href="javascript:;" ng-click="$ctrl.popTocMenu($event, $ctrl.panel)">
            <i class="wtf {{$ctrl.panel | panelIconFont}}"></i>{{$ctrl.panel.name}}
        </a>
    </div>
    <div class="flex-panel">
        <button type="button" class="btn btn-primary" ng-if="$ctrl.panel.privileges | hasPortalPrivilege:'editPanel'" ng-click="$ctrl.openSelectWidget()"><i class="wtf wtf-th-plus"></i> {{'portal.ADD_WIDGET_BUTTON'|translate}}</button>
    </div>
</div>

<div class="module-body">
    <part-loading done="$ctrl.loadingDone"></part-loading>
    <div class="module-body-content" ng-if="$ctrl.loadingDone && (!$ctrl.panel || $ctrl.portalService.widgets.length === 0)">
        <empty-state entity-name="部件" size="lg"></empty-state>
    </div>
    <div id="portal-layout" class="portal-layout-wrap">

        <div id="portal-layout-container"
             class="portal-layout-container free-wall" ng-show="$ctrl.loadingDone">
            <div class="widget-wrap widget-item-{{widget._id}}" ng-class="{'widget-wrap--sm' : $ctrl.isSmallWidget}"
                widget-id="{{widget._id}}"
                ng-init="widget.isSmallWidget = $ctrl.isSmallWidget"
                ng-repeat="widget in $ctrl.portalService.widgets track by widget._id"
                ng-style="widget.style" data-width="{{widget.style.width}}" data-height="{{widget.style.height}}"
                data-handle=".handle">
                <portal-widget-switch widget="widget"></portal-widget-switch>
            </div>
        </div>
    </div>
</div>







.portal-layout-wrap {
  .portal-layout-container {
    .widget-wrap {
      height: 100%;
      .panel {
        height: 100%;
        display: flex;
        flex-direction: column;
        .panel-body {
          overflow-y: auto;
          flex: 1;
          &--widget-personal-card {
            padding: 5% 0;
            margin: 0 auto;
            overflow: hidden;
          }
        }
      }
    }
    //.widget-item {
    //  height: 100%;
    //  > div {
    //    height: 100%;
    //    display: flex;
    //    flex-direction: column;
    //  }
    //}
    //.portal-item {
    //  //border: 1px @border-color solid;
    //  margin-bottom: 10px;
    //  border-radius: 5px;
    //  padding: 0 15px;
    //  background: @bg-color-content;
    //  //box-shadow: 0 0 5px rgba(0, 0, 0, 0.2);
    //  //padding-bottom: 10px;
    //  .portal-header {
    //    background: none;
    //    padding: 15px 0px;
    //    .handle {
    //      cursor: move;
    //    }
    //  }
    //  .portal-body {
    //    border-top: 1px @color-border solid;
    //    padding: 15px 0;
    //    overflow-y: auto;
    //    flex: 1;
    //  }
    //}
  }
}

```
